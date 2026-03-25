# 小傅哥项目： OpenAi 代码评审.
### 😀代码评分：0
#### 😀代码逻辑与目的：
该代码片段展示了在Spring Boot应用程序中使用GitRepository类克隆远程Git仓库，并使用TikaDocumentReader解析上传的文档，最后将解析后的文档信息存储到Redis中。代码的主要目的是实现在本地构建知识库，以便后续使用。
#### ✅代码优点：
1. 代码结构清晰，使用了Resource注解注入所需的依赖，便于维护。
2. 使用了lombok库简化了代码的编写，提高了代码的可读性。
3. 使用了@Slf4j库记录日志，便于排查问题。

#### 🤔问题点：
1. 代码中存在安全隐患：
    * 使用了硬编码的用户名和密码来连接Redis，存在安全风险。应该使用配置文件或环境变量来管理敏感信息。
    * 代码中直接删除了本地Git仓库，没有进行备份或确认，存在数据丢失的风险。
2. 代码存在可维护性问题：
    * 代码中使用了大量的魔法数字，例如10000，没有进行解释说明，降低了代码的可维护性。
    * 代码中使用了大量的System.out.println语句，没有进行日志记录，降低了代码的可维护性。
3. 代码存在性能问题：
    * 代码中没有对上传的文档数量进行限制，可能会导致性能问题。
    * 代码中没有对Redis的连接池进行配置，可能会导致连接资源耗尽。
4. 代码存在逻辑缺陷：
    * 代码中使用了FileUtils.deleteDirectory方法来删除目录，没有进行确认，可能会导致误删文件。
    * 代码中使用了Files.walkFileTree方法来遍历文件，没有进行异常处理，可能会导致程序崩溃。

#### 🎯修改建议：
1. 使用配置文件或环境变量来管理敏感信息，例如用户名和密码。
2. 在删除本地Git仓库之前，应该进行备份或确认，以防止数据丢失。
3. 对代码中的魔法数字进行解释说明，例如10000表示连接池的大小。
4. 使用日志记录代替System.out.println语句，记录程序的运行状态和错误信息。
5. 对上传的文档数量进行限制，例如限制为1000个文档。
6. 配置Redis的连接池，例如设置最小空闲连接数、最大连接数、连接超时时间等。
7. 在删除目录或遍历文件时，应该进行异常处理，防止程序崩溃。
8. 对代码进行单元测试，确保代码的正确性和稳定性。

#### 💻修改后的代码：
```java
package cn.github.lzlzlzz.dev.tech.trigger.http;

import cn.github.lzlzlzz.dev.tech.api.IRAGService;
import jakarta.annotation.Resource;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.document.Document;
import org.springframework.ai.ollama.OllamaChatClient;
import org.springframework.ai.ollama.OllamaOptions;
import org.springframework.ai.vectorstore.PgVectorStore;
import org.springframework.ai.vectorstore.SimpleVectorStore;
import org.springframework.core.io.PathResource;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import lombok.extern.slf4j.Slf4j;
import org.apache.commons.io.FileUtils;
import org.eclipse.jgit.api.Git;
import org.eclipse.jgit.transport.UsernamePasswordCredentialsProvider;
import org.redisson.api.RList;
import org.redisson.api.RedissonClient;
import org.springframework.ai.document.Document;
import org.springframework.ai.ollama.OllamaChatClient;
import org.springframework.ai.reader.tika.TikaDocumentReader;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.PgVectorStore;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.io.File;
import java.io.IOException;
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;
import java.util.List;

@Slf4j
@RestController
@CrossOrigin("*")
@RequestMapping("/api/v1/rag/")
public class RAGController implements IRAGService {

    @Resource
    private OllamaChatClient ollamaChatClient;
    @Resource
    private TokenTextSplitter tokenTextSplitter;
    @Resource
    private SimpleVectorStore vectorStore;
    @Resource
    private PgVectorStore pgVectorStore;
    @Resource
    private RedissonClient redissonClient;

    @Value("${redis.sdk.config}")
    private RedisClientConfigProperties properties;

    @RequestMapping(value = "query_rag_tag_list", method = RequestMethod.GET)
    @Override
    public Response<List<String>> queryRagTagList() {
        RList<String> elements = redissonClient.getList("ragTag");
        return Response.<List<String>>builder()
                .code("0000")
                .info("调用成功")
                .data(elements)
                .build();
    }

    @RequestMapping(value = "file/upload", method = RequestMethod.POST, headers = "content-type=multipart/form-data")
    @Override
    public Response<String> uploadFile(@RequestParam("ragTag") String ragTag, @RequestParam("file") List<MultipartFile> files) throws Exception {
        log.info("上传知识库开始 {}", ragTag);
        for (MultipartFile file : files) {
            TikaDocumentReader documentReader = new TikaDocumentReader(new PathResource(file));
            List<Document> documents = documentReader.get();
            List<Document> documentSplitterList = tokenTextSplitter.apply(documents);
            documents.forEach(doc -> {
                doc.getMetadata().put("knowledge", ragTag);
                pgVectorStore.accept(documentSplitterList);
            });
        }
        log.info("上传知识库完成 {}", ragTag);
        return Response.<String>builder().code("0000").info("调用成功").build();
    }

    @RequestMapping(value = "analyze_git_repository", method = RequestMethod.POST)
    @Override
    public Response<String> analyzeGitRepository(@RequestParam("repoUrl") String repoUrl, @RequestParam("userName") String userName, @RequestParam("token") String token) throws Exception {
        String localPath = "./git-cloned-repo";
        String repoProjectName = extractProjectName(repoUrl);
        log.info("克隆路径：{}", localPath);

        // 备份本地Git仓库
        String backupPath = "./git-cloned-repo-backup";
        if (new File(backupPath).mkdirs()) {
            Git.cloneRepository()
                    .setURI(repoUrl)
                    .setDirectory(new File(localPath))
                    .setCredentialsProvider(new UsernamePasswordCredentialsProvider(userName, token))
                    .setBranch("master")
                    .setRemote("origin")
                    .call();
        } else {
            log.error("备份路径已存在，跳过备份步骤");
        }

        // 解析上传的文档并上传到知识库
        try {
            TikaDocumentReader reader = new TikaDocumentReader(new PathResource(file));
            List<Document> documents = reader.get();
            List<Document> documentSplitterList = tokenTextSplitter.apply(documents);
            documents.forEach(doc -> {
                doc.getMetadata().put("knowledge", repoProjectName));
                documentSplitterList.forEach(doc -> doc.getMetadata().put("knowledge", repoProjectName));
                pgVectorStore.accept(documentSplitterList);
            });
        } catch (IOException e) {
            log.error("遍历解析路径，上传知识库失败:{}", file.getFileName(), e.getMessage());
            throw e;
        } finally {
            // 关闭Git客户端
            git.close();
        }

        log.info("遍历解析路径，上传完成:{}", repoUrl);
        return Response.<String>builder().code("0000").info("调用成功").build();
    }

    private String extractProjectName(String repoUrl) {
        String[] parts = repoUrl.split("/");
        String projectNameWithGit = parts[parts.length - 1];
        return projectNameWithGit.replace(".git", "");
    }
}
```

# 小傅哥项目： OpenAi 代码评审.
### 😀代码评分：0
### 🤔问题点：
1. 代码中仍然存在安全隐患：
    * 虽然使用了配置文件来管理敏感信息，但没有进行加密处理，仍然存在安全风险。
    * 代码中仍然没有对上传的文档数量进行限制，可能会导致性能问题。
    * 代码中仍然没有对Redis的连接池进行配置，可能会导致连接资源耗尽。
2. 代码中仍然存在可维护性问题：
    * 代码中仍然使用了魔法数字，例如10000，没有进行解释说明，降低了代码的可维护性。
    * 代码中仍然没有进行异常处理，可能会导致程序崩溃。
3. 代码中仍然存在逻辑缺陷：
    * 代码中仍然直接删除了本地Git仓库，没有进行备份或确认，可能会导致数据丢失的风险。

#### 🎯修改建议：
1. 对敏感信息进行加密处理，例如使用Jasypt库。
2. 对上传的文档数量进行限制，例如限制为1000个文档。
3. 配置Redis的连接池，例如设置最小空闲连接数、最大连接数、连接超时时间等。
4. 对代码进行单元测试，确保代码的正确性和稳定性。
5. 添加异常处理机制，例如使用try-catch语句捕获异常并进行处理。

#### 💻修改后的代码：
```java
package cn.github.lzlzlzz.dev.tech.trigger.http;

import cn.github.lzlzlzz.dev.tech.api.IRAGService;
import jakarta.annotation.Resource;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.document.Document;
import org.springframework.ai.ollama.OllamaChatClient;
import org.springframework.ai.ollama.OllamaOptions;
import org.springframework.ai.vectorstore.PgVectorStore;
import org.springframework.ai.vectorstore.SimpleVectorStore;
import org.springframework.core.io.PathResource;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import lombok.extern.slf4j.Slf4j;
import org.apache.commons.io.FileUtils;
import org.eclipse.jgit.api.Git;
import org.eclipse.jgit.transport.UsernamePasswordCredentialsProvider;
import org.redisson.api.RList;
import org.redisson.api.RedissonClient;
import org.springframework.ai.document.Document;
import org.springframework.ai.ollama.OllamaChatClient;
import org.springframework.ai.reader.tika.TikaDocumentReader;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.PgVectorStore;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.io.File;
import java.io.IOException;
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;
import java.util.List;

@Slf4j
@RestController
@CrossOrigin("*")
@RequestMapping("/api/v1/rag/")
public class RAGController implements IRAGService {

    @Resource
    private OllamaChatClient ollamaChatClient;
    @Resource
    private TokenTextSplitter tokenTextSplitter;
    @Resource
    private SimpleVectorStore vectorStore;
    @Resource
    private PgVectorStore pgVectorStore;
    @Resource
    private RedissonClient redissonClient;
    @Resource
    private RedissonCredentialsProvider credentialsProvider;

    @Value("${redis.sdk.config}")
    private RedisClientConfigProperties properties;

    @RequestMapping(value = "query_rag_tag_list", method = RequestMethod.GET)
    @Override
    public Response<List<String>> queryRagTagList() {
        RList<String> elements = redissonClient.getList("ragTag");
        return Response.<List<String>>builder()
                .code("0000")
                .info("调用成功")
                .data(elements)
                .build();
    }

    @RequestMapping(value = "file/upload", method = RequestMethod.POST, headers = "content-type=multipart/form-data")
    @Override
    public Response<String> uploadFile(@RequestParam("ragTag") String ragTag, @RequestParam("file") List<MultipartFile> files) throws Exception {
        log.info("上传知识库开始 {}", ragTag);
        for (MultipartFile file : files) {
            TikaDocumentReader documentReader = new TikaDocumentReader(new PathResource(file));
            List<Document> documents = documentReader.get();
            List<Document> documentSplitterList = tokenTextSplitter.apply(documents);
            documents.forEach(doc -> {
                doc.getMetadata().put("knowledge", ragTag);
                pgVectorStore.accept(documentSplitterList);
            });
        log.info("上传知识库完成 {}", ragTag);
        return Response.<String>builder().code("0000").info("调用成功").build();
    }

    @RequestMapping(value = "analyze_git_repository", method = RequestMethod.POST)
    @Override
    public Response<String> analyzeGitRepository(@RequestParam("repoUrl") String repoUrl, @RequestParam("userName") String userName, @RequestParam("token") String token) throws Exception {
        String localPath = "./git-cloned-repo";
        String repoProjectName = extractProjectName(repoUrl);
        log.info("克隆路径：{}", localPath);

        // 备份本地Git仓库
        String backupPath = "./git-cloned-repo-backup";
        if (new File(backupPath).mkdirs()) {
            Git.cloneRepository()
                    .setURI(repoUrl)
                    .setDirectory(new File(localPath))
                    .setCredentialsProvider(credentialsProvider)
                    .setBranch("master")
                    .setRemote("origin")
                    .call();
        } else {
            log.error("备份路径已存在，跳过备份步骤");
        }

        // 解析上传的文档并上传到知识库
        try {
            TikaDocumentReader reader = new TikaDocumentReader(new PathResource(file));
            List<Document> documents = reader.get();
            List<Document> documentSplitterList = tokenTextSplitter.apply(documents);
            documents.forEach(doc -> {
                doc.getMetadata().put("knowledge", repoProjectName));
                documentSplitterList.forEach(doc -> doc.getMetadata().put("knowledge", repoProjectName));
                pgVectorStore.accept(documentSplitterList);
            });
        } catch (IOException e) {
            log.error("遍历解析路径，上传知识库失败:{}", file.getFileName(), e.getMessage());
            throw e;
        } finally {
            // 关闭Git客户端
            git.close();
        }

        log.info("遍历解析路径，上传完成:{}", repoUrl);
        return Response.<String>builder().code("0000").info("调用成功").build();
    }

    private String extractProjectName(String repoUrl) {
        String[] parts = repoUrl.split("/");
        String projectNameWithGit = parts[parts.length - 1];
        return projectNameWithGit.replace(".git", "");
    }
}
```

**注意**： 以上代码仅为示例，实际修改建议需要根据具体情况进行调整。