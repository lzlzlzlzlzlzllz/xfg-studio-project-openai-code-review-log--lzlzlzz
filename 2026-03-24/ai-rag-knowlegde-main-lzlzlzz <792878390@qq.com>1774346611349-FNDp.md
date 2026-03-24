# 小傅哥项目： OpenAi 代码评审.
### 😀代码评分：95
#### 😀代码逻辑与目的：
本代码定义了两个 GitHub Actions 工作流：`main.yml` 和 `qodana_code_quality.yml`。`main.yml` 工作流用于在代码推送或 PR 时执行代码评审，包括检出仓库、下载已发布的 SDK JAR、注入环境变量并执行 Java 程序。`qodana_code_quality.yml` 工作流用于在代码推送、PR 或手动触发时执行代码质量分析。此外，还创建了 `.idea` 目录下的配置文件，用于设置项目编码和版本控制。
#### ✅代码优点：
1. **结构清晰**：工作流文件结构清晰，步骤明确。
2. **环境隔离**：使用 `actions/checkout@v2` 和 `actions/setup-java@v2` 确保环境隔离和一致性。
3. **日志记录**：记录了仓库名、分支名、提交作者和提交信息，便于追踪。
4. **安全性**：使用 GitHub Secrets 管理敏感信息，如 API 密钥和微信配置。
5. **代码质量**：使用 Qodana 进行代码质量分析，确保代码质量。
#### 🤔问题点：
1. **依赖管理**：直接下载 SDK JAR，缺乏版本控制和依赖管理。
2. **环境变量**：环境变量注入较多，需确保其安全性。
3. **错误处理**：未明确处理 SDK 运行失败的情况。
4. **资源清理**：下载的 JAR 文件未在运行后清理，可能导致资源浪费。
5. **Qodana 配置**：`qodana.yaml` 文件中部分配置项未使用，需明确其用途。
#### 🎯修改建议：
1. **依赖管理**：使用 Maven 或 Gradle 管理依赖，避免直接下载 JAR 文件。
2. **环境变量**：减少环境变量数量，确保敏感信息的安全性。
3. **错误处理**：增加对 SDK 运行失败的错误处理逻辑。
4. **资源清理**：在工作流中增加步骤清理下载的 JAR 文件。
5. **Qodana 配置**：明确 `qodana.yaml` 文件中未使用配置项的用途，避免冗余。
#### 💻修改后的代码：
```yaml
diff --git a/.github/workflows/main.yml b/.github/workflows/main.yml
index 04b49b2..new_index
--- a/.github/workflows/main.yml
+++ b/.github/workflows/main.yml
@@ -1,80 +1,80 @@
+# 【文件作用】push/PR 时检出仓库、下载已发布的 SDK JAR、注入环境变量并执行 java -jar，完成一次代码评审。
+# 【所属功能】CI/CD - 工作流（远程 JAR，任意分支）
+name: OpenAiCodeReview
+
+on:
+  # 所有分支 push 时都触发自动评审
+  push:
+    branches:
+      - '*'
+  # PR 也触发，便于在合并前给出评审建议
+  pull_request:
+    branches:
+      - '*'
+
+jobs:
+  build:
+    runs-on: ubuntu-latest
+
+    steps:
+      - name: Checkout repository
+        uses: actions/checkout@v2
+        with:
+          # 至少拉两个提交，后面才能比较“最近一次提交”和“它的前一个提交”
+          fetch-depth: 2  # 检出最后两个提交，以便可以比较 HEAD~1 和 HEAD
+
+      - name: Set up JDK 11
+        uses: actions/setup-java@v2
+        with:
+          distribution: 'adopt'
+          java-version: '11'
+
+      - name: Create libs directory
+        run: mkdir -p ./libs
+
+      - name: Download openai-code-review-sdk JAR
+        # 直接下载发布好的 SDK JAR，避免每次工作流都重新打包
+        run: wget -O ./libs/openai-code-review-sdk-1.0.jar https://github.com/lzlzlzlzlzlzllz/xfg-studio-project-openai-code-review-log--lzlzlzz/releases/download/v1.0/openai-code-review-sdk-1.0.jar
+
+      - name: Get repository name
+        id: repo-name
+        run: echo "REPO_NAME=${GITHUB_REPOSITORY##*/}" >> $GITHUB_ENV
+
+      - name: Get branch name
+        id: branch-name
+        run: echo "BRANCH_NAME=${GITHUB_REF#refs/heads/}" >> $GITHUB_ENV
+
+      - name: Get commit author
+        id: commit-author
+        run: echo "COMMIT_AUTHOR=$(git log -1 --pretty=format:'%an <%ae>')" >> $GITHUB_ENV
+
+      - name: Get commit message
+        id: commit-message
+        run: echo "COMMIT_MESSAGE=$(git log -1 --pretty=format:'%s')" >> $GITHUB_ENV
+
+      - name: Print repository, branch name, commit author, and commit message
+        run: |
+          echo "Repository name is ${{ env.REPO_NAME }}"
+          echo "Branch name is ${{ env.BRANCH_NAME }}"
+          echo "Commit author is ${{ env.COMMIT_AUTHOR }}"
+          echo "Commit message is ${{ env.COMMIT_MESSAGE }}"
+
+      - name: Run Code Review
+        # SDK 会在运行过程中自动完成：取 diff -> AI 评审 -> 提交日志 -> 微信通知
+        run: java -jar ./libs/openai-code-review-sdk-1.0.jar
+        env:
+          # Github 配置；GITHUB_REVIEW_LOG_URI「https://github.com/xfg-studio-project/openai-code-review-log-lzlzlzz」、GITHUB_TOKEN「https://github.com/settings/tokens」
+          GITHUB_REVIEW_LOG_URI: ${{ secrets.CODE_REVIEW_LOG_URL }}
+          GITHUB_TOKEN: ${{ secrets.CODE_TOKEN }}
+          COMMIT_PROJECT: ${{ env.REPO_NAME }}
+          COMMIT_BRANCH: ${{ env.BRANCH_NAME }}
+          COMMIT_AUTHOR: ${{ env.COMMIT_AUTHOR }}
+          COMMIT_MESSAGE: ${{ env.COMMIT_MESSAGE }}
+          # 微信配置 「https://mp.weixin.qq.com/debug/cgi-bin/sandboxinfo?action=showinfo&t=sandbox/index」
+          WEIXIN_APPID: ${{ secrets.WEIXIN_APPID }}
+          WEIXIN_SECRET: ${{ secrets.WEIXIN_SECRET }}
+          WEIXIN_TOUSER: ${{ secrets.WEIXIN_TOUSER }}
+          WEIXIN_TEMPLATE_ID: ${{ secrets.WEIXIN_TEMPLATE_ID }}
+          # OpenAi - ChatGLM 配置「https://open.bigmodel.cn/api/paas/v4/chat/completions」、「https://open.bigmodel.cn/usercenter/apikeys」
+          CHATGLM_APIHOST: ${{ secrets.CHATGLM_APIHOST }}
+          CHATGLM_APIKEYSECRET: ${{ secrets.CHATGLM_APIKEYSECRET }}
+
+      - name: Clean up downloaded JAR
+        run: rm -f ./libs/openai-code-review-sdk-1.0.jar
diff --git a/.github/workflows/qodana_code_quality.yml b/.github/workflows/qodana_code_quality.yml
index fb4f3e1..new_index
--- a/.github/workflows/qodana_code_quality.yml
+++ b/.github/workflows/qodana_code_quality.yml
@@ -1,39 +1,39 @@
+#-------------------------------------------------------------------------------#
+#            Discover all capabilities of Qodana in our documentation           #
+#             https://www.jetbrains.com/help/qodana/about-qodana.html           #
+#-------------------------------------------------------------------------------#
+
+name: Qodana
+on:
+  workflow_dispatch:
+  pull_request:
+  push:
+    branches:
+      - main
+
+jobs:
+  qodana:
+    runs-on: ubuntu-latest
+    permissions:
+      contents: write
+      pull-requests: write
+      checks: write
+    steps:
+      - uses: actions/checkout@v4
+        with:
+          ref: ${{ github.event.pull_request.head.sha }}
+          fetch-depth: 0
+      - name: 'Qodana Scan'
+        uses: JetBrains/qodana-action@v2025.3
+        env:
+          QODANA_TOKEN: ${{ secrets.QODANA_TOKEN }}
+        with:
+          # When pr-mode is set to true, Qodana analyzes only the files that have been changed
+          pr-mode: false
+          use-caches: true
+          post-pr-comment: true
+          use-annotations: true
+          # Upload Qodana results (SARIF, other artifacts, logs) as an artifact to the job
+          upload-result: false
+          # quick-fixes available in Ultimate and Ultimate Plus plans
+          push-fixes: 'none'
diff --git a/.idea/encodings.xml b/.idea/encodings.xml
index c0eb66d..new_index
--- a/.idea/encodings.xml
+++ b/.idea/encodings.xml
@@ -1,17 +1,17 @@
 <?xml version="1.0" encoding="UTF-8"?>
 <project version="4">
   <component name="Encoding">
     <file url="file://$PROJECT_DIR$/lzlzlzz-dev-tech-api/src/main/java" charset="UTF-8" />
     <file url="file://$PROJECT_DIR$/lzlzlzz-dev-tech-api/src/main/resources" charset="UTF-8" />
     <file url="file://$PROJECT_DIR$/lzlzlzz-dev-tech-app/src/main/java" charset="UTF-8" />
     <file url="file://$PROJECT_DIR$/lzlzlzz-dev-tech-app/src/main/resources" charset="UTF-8" />
     <file url="file://$PROJECT_DIR$/lzlzlzz-dev-tech-trigger/src/main/java" charset="UTF-8" />
     <file url="file://$PROJECT_DIR$/lzlzlzz-dev-tech-trigger/src/main/resources" charset="UTF-8" />
     <file url="file://$PROJECT_DIR$/lzlzlzz-rag-tech-app/src/main/java" charset="UTF-8" />
     <file url="file://$PROJECT_DIR$/lzlzlzz-rag-tech-app/src/main/resources" charset="UTF-8" />
     <file url="file://$PROJECT_DIR$/src/main/java" charset="UTF-8" />
     <file url="file://$PROJECT_DIR$/src/main/resources" charset="UTF-8" />
     <file url="file:///src/main/java" charset="UTF-8" />
     <file url="file:///src/main/resources" charset="UTF-8" />
   </component>
 </project>
diff --git a/.idea/misc.xml b/.idea/misc.xml
index 2f30dbf..new_index
--- a/.idea/misc.xml
+++ b/.idea/misc.xml
@@ -1,14 +1,14 @@
 <?xml version="1.0" encoding="UTF-8"?>
 <project version="4">
   <component name="ExternalStorageConfigurationManager" enabled="true" />
   <component name="MavenProjectsManager">
     <option name="originalFiles">
       <list>
         <option value="$PROJECT_DIR$/pom.xml" />
       </list>
     </option>
   </component>
   <component name="ProjectRootManager" version="2" languageLevel="JDK_17" default="true" project-jdk-name="ms-17" project-jdk-type="JavaSDK">
     <output url="file://$PROJECT_DIR$/out" />
   </component>
 </project>
diff --git a/.idea/vcs.xml b/.idea/vcs.xml
index 94a25f7..new_index
--- a/.idea/vcs.xml
+++ b/.idea/vcs.xml
@@ -1,6 +1,6 @@
 <?xml version="1.0" encoding="UTF-8"?>
 <project version="4">
   <component name="VcsDirectoryMappings">
     <mapping directory="$PROJECT_DIR$" vcs="Git" />
   </component>
 </project>
diff --git a/.idea/workspace.xml b/.idea/workspace.xml
index 3d509a1..new_index
--- a/.idea/workspace.xml
+++ b/.idea/workspace.xml
@@ -1,139 +1,139 @@
 <?xml version="1.0" encoding="UTF-8"?>
 <project version="4">
   <component name="AutoImportSettings">
     <option name="autoReloadType" value="SELECTIVE" />
   </component>
   <component name="ChangeListManager">
     <list default="true" id="4be2e12b-72ed-4be3-940b-ff1b628cf9cf" name="Changes" comment="">
       <change afterPath="$PROJECT_DIR$/.gitignore" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/api/curl.json" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/api/curl.sh" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/docker-compose-environment-aliyun.yml" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/docker-compose-environment.yml" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/nginx/html/css/index.css" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/nginx/html/git.html" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/nginx/html/images/ai-rag-knowledge-6-01.png" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/nginx/html/index.html" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/nginx/html/js/index.js" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/nginx/html/upload.html" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/nginx/html/第4节：ai-case-04.html" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/nginx/html/第7节：ai-case-01.html" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/nginx/html/第7节：ai-case-02.html" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/nginx/html/第7节：ai-case-03.html" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/pgvector/sql/init.sql" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/dev-ops/redis/redis.conf" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/lzlzlzz-dev-tech-api/pom.xml" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/lzlzlzz-dev-tech-api/src/main/java/cn/github/lzlzlzz/dev/tech/api/package-info.java" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/lzlzlzz-dev-tech-app/pom.xml" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/lzlzlzz-dev-tech-app/src/main/java/cn/github/lzlzlzz/dev/tech/package-info.java" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/lzlzlzz-dev-tech-trigger/pom.xml" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/lzlzlzz-dev-tech-trigger/src/main/java/cn/github/dev/tech/trigger/package-info.java" afterDir="false" />
       <change afterPath="$PROJECT_DIR$/pom.xml" afterDir="false" />
     </list>
     <option name="SHOW_DIALOG" value="false" />
     <option name="HIGHLIGHT_CONFLICTS" value="true" />
     <option name="HIGHLIGHT_NON_ACTIVE_CHANGELIST" value="false" />
     <option name="LAST_RESOLUTION" value="IGNORE" />
   </component>
   <component name="FileTemplateManagerImpl">
     <option name="RECENT_TEMPLATES">
       <list>
         <option value="package-info" />
       </list>
     </option>
   </component>
   <component name="Git.Settings">
     <option name="RECENT_GIT_ROOT_PATH" value="$PROJECT_DIR$" />
   </component>
   <component name="GithubPullRequestsUISettings">{
   "selectedUrlAndAccountId": {
     "url": "https://github.com/lzlzlzz/ai-rag-knowlegde.git",
     "accountId": "2e05bd52-aedc-4146-8ccd-85f82ad0ec0f"
   }
 }</component>
   <component name="ProjectColorInfo">{
   "associatedIndex": 7
 }</component>
   <component name="ProjectId" id="3BNrS1e5D2dB9giAY3vi72CMP3Y" />
   <component name="ProjectViewState">
     <option name="hideEmptyMiddlePackages" value="true" />
     <option name="showLibraryContents" value="true" />
     <option name="showScratchesAndConsoles" value="false" />
   </component>
   <component name="PropertiesComponent">{
   "keyToString": {
     "ASKED_SHARE_PROJECT_CONFIGURATION_FILES": "true",
     "Maven.ai-rag-knowledge [clean].executor": "Run",
     "ModuleVcsDetector.initialDetectionPerformed": "true",
     "RunOnceActivity.ShowReadmeOnStart": "true",
     "RunOnceActivity.typescript.service.memoryLimit.init": "true",
     "SHELLCHECK.PATH": "/Users/lzlzlzz/Library/Application Support/JetBrains/IntelliJIdea2025.3/plugins/Shell Script/shellcheck",
     "git-widget-placeholder": "main",
     "kotlin-language-version-configured": "true",
     "nodejs_package_manager_path": "npm",
     "onboarding.tips.debug.path": "/Users/lzlzlzz/Documents/xfg_agent/ai-rag-knowledge/ai-rag-knowledge/src/main/java/com/lz/middleware/Main.java",
     "project.structure.last.edited": "Modules",
     "project.structure.proportion": "0.0",
     "project.structure.side.proportion": "0.0",
     "settings.editor.selected.configurable": "vcs.Git",
     "vue.rearranger.settings.migration": "true"
   }
 }</component>
   <component name="QodanaGithubNotificationPromoProjectDismissalState">
     <option name="dismissed" value="true" />
   </component>
   <component name="RecentsManager">
     <key name="MoveFile.RECENT_KEYS">
       <recent name="$PROJECT_DIR$/dev-ops" />
       <recent name="$PROJECT_DIR$/dev-ous" />
     </key>
   </component>
   <component name="SharedIndexes">
     <attachedChunks>
       <set>
         <option value="bundled-jdk-30f59d01ecdd-cffe25b9f5b3-intellij.indexing.shared.core-IU-253.28294.334" />
         <option value="bundled-js-predefined-d6986cc7102b-c7e53b3be11b-JavaScript-IU-253.28294.334" />
       </set>
     </attachedChunks>
   </component>
   <component name="TaskManager">
     <task active="true" id="Default" summary="Default task">
       <changelist id="4be2e12b-72ed-4be3-940b-ff1b628cf9cf" name="Changes" comment="" />
       <created>1774340364536</created>
       <option name="number" value="Default" />
       <option name="presentableId" value="Default" />
       <updated>1774340364536</updated>
       <workItem from="1774340365680" duration="4536000" />
     </task>
     <servers />
   </component>
   <component name="TypeScriptGeneratedFilesManager">
     <option name="version" value="3" />
   </component>
   <component name="Vcs.Log.Tabs.Properties">
     <option name="TAB_STATES">
       <map>
         <entry key="MAIN">
           <value>
             <State />
           </value>
         </entry>
       </map>
     </option>
   </component>
   <component name="XDebuggerManager">
     <breakpoint-manager>
       <breakpoints>
         <line-breakpoint enabled="true" type="java-line">
           <url>file://$PROJECT_DIR$/src/main/java/com/lz/middleware/Main.java</url>
           <line>13</line>
           <option name="timeStamp" value="1" />
         </line-breakpoint>
       </breakpoints>
     </breakpoint-manager>
   </component>
   <component name="XSLT-Support.FileAssociations.UIState">
     <expand />
     <select />
   </component>
 </project>
```
```yaml
diff --git a/qodana.yaml b/qodana.yaml
index ddc1568..new_index
--- a/qodana.yaml
+++ b/qodana.yaml
@@ -1,49 +1,49 @@
+#-------------------------------------------------------------------------------#
+#               Qodana analysis is configured by qodana.yaml file               #
+#             https://www.jetbrains.com/help/qodana/qodana-yaml.html            #
+#-------------------------------------------------------------------------------#
+
+#################################################################################
+#              WARNING: Do not store sensitive information in this file,        #
+#               as its contents will be included in the Qodana report.          #
+#################################################################################
+version: "1.0"
+
+#Specify inspection profile for code analysis
+profile:
+  name: qodana.starter
+
+#Enable inspections
+#include:
+#  - name: <SomeEnabledInspectionId>
+
+#Disable inspections
+#exclude:
+#  - name: <SomeDisabledInspectionId>
+#    paths:
+#      - <path/where/not/run/inspection>
+
+projectJDK: "17" #(Applied in CI/CD pipeline)
+
+#Execute shell command before Qodana execution (Applied in CI/CD pipeline)
+#bootstrap: sh ./prepare-qodana.sh
+
+#Install IDE plugins before Qodana execution (Applied in CI/CD pipeline)
+#plugins:
+#  - id: <plugin.id> #(plugin id can be found at https://plugins.jetbrains.com)
+
+# Quality gate. Will fail the CI/CD pipeline if any condition is not met
+# severityThresholds - configures maximum thresholds for different problem severities
+# testCoverageThresholds - configures minimum code coverage on a whole project and newly added code
+# Code Coverage is available in Ultimate and Ultimate Plus plans
+#failureConditions:
+#  severityThresholds:
+#    any: 15
+#    critical: 5
+#  testCoverageThresholds:
+#    fresh: 70
+#    total: 50
+  
+#Qodana supports other languages, for example, Python, JavaScript, TypeScript, Go, C#, PHP
+#For all supported languages see https://www.jetbrains.com/help/qodana/linters.html
+linter: jetbrains/qodana-jvm-community:2025.3
```