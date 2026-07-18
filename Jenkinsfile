pipeline {
    agent {
        kubernetes {
            label "prefix-bench-${UUID.randomUUID().toString()}"
            defaultContainer 'aisbench'
            yaml getPodYaml()
        }
    }

    parameters {
        string(name: 'SERVER_URL', defaultValue: '141.111.32.72:8000',
               description: '推理服务地址，格式为 IP:PORT 或完整 URL（Docker场景用 http://host.docker.internal:8080）')
        string(name: 'API_MODEL_NAME', defaultValue: 'Qwen3-1.7B',
               description: '服务端已加载的模型名称，依据实际 vLLM 推理服务配置填写')
        activeChoice(
            name: 'MODEL_FOLDER_NAME',
            choiceType: 'PT_SINGLE_SELECT',
            description: '共享存储中的模型路径，挂载点是 /mnt/model/',
            filterable: true,
            filterLength: 2,
            script: [
              $class: 'GroovyScript',
              script: [
                classpath: [],
                sandbox: false,
                script: '''
                  def myTargetFolder ='/mnt/model/'
                  def root = new File(myTargetFolder)
                  if (!root.isDirectory() || !root.canRead()) {
                    return [('/models/(folder not found or not readable: ' + myTargetFolder + ')') as String]
                  }
                  def choices = []
                  root.eachDir() { dir ->
                    choices << dir.name
                  }
                  choices.sort()
                  return choices as String
                  '''
              ],
              fallbackScript: [
                classpath: [],
                sandbox: false,
                script: 'return ["/models/(error listing /mnt/models)"]'
              ]
            ]
          )
        choice(name: 'CUSTOM_TEST_IMAGE',
            choices: [
              'registry.dev.huawei.com/flash_stor/aisbench_benchmark:v3.1_x86_64_py_310',
              'registry.dev.huawei.com/flash_stor/aisbench_benchmark:v3.1_x86_64_py_310-transformersv5'
            ],
            description: '执行测试使用的镜像')

        text(name: 'TEST_CASES', defaultValue: '''[
    {
        "input_len": 3500,
        "output_len": 1500,
        "data_num": 8192,
        "concurrency": 2048,
        "request_rate": 0,
        "prefix_num": 1,
        "repeat_rate": 0.5,
        "seed": 1
    },
    {
        "input_len": 8500,
        "output_len": 1500,
        "data_num": 8192,
        "concurrency": 2048,
        "request_rate": 0,
        "prefix_num": 1,
        "repeat_rate": 0.5,
        "seed": 2
    }
]''',
            description: '''多轮测试参数（JSON 数组），每项是一个 dict 覆盖本轮配置。
可覆盖字段：input_len, output_len, data_num, concurrency, request_rate, prefix_num, repeat_rate, seed, dp, test_name。
缺省字段继承 config.py 默认值。seed=0 表示纯随机。
示例：
  [{"input_len":3500,"seed":1},{"input_len":8500,"concurrency":4096,"seed":0}]
  [{"test_name":"short_prefix","input_len":2000,"repeat_rate":"50%","dp":2}]
''')

        string(name: 'DP', defaultValue: '1',
               description: '数据并行度（Data-Parallelism），影响 warmup 并发数')
        string(name: 'NPU_NUM', defaultValue: '1',
               description: 'NPU 卡数，用于单卡吞吐计算')
    }

    options {
        ansiColor('xterm')
        timestamps()
        copyArtifactPermission('*')
    }

    environment {
        VALUES_REPO        = "ssh://git@szv-open.codehub.huawei.com:2222/innersource/UnifiedCache_G/Contribute.git"
        VALUES_REPO_BRANCH = "personal_jenkins_dxlong"
        VALUES_REPO_DIR    = "Jenkins-K8s/aisbench_auto_tools_prefix-main"
        REPO_KEY           = "jenkins-ci-ssh"
    }

    stages {
        stage("clone repo") {
            steps {
                container('aisbench') {
                    script {
                        withCredentials([sshUserPrivateKey(
                            keyFileVariable: 'SSH_KEY',
                            usernameVariable: 'SSH_USER',
                            credentialsId: env.REPO_KEY
                        )]) {
                            sh '''
                                set -x
                                rm -rf aisbench_auto_tools_prefix-main
                                git -c core.sshCommand="ssh -i $SSH_KEY -o StrictHostKeyChecking=no -p 2222" \
                                    clone -v -b "$VALUES_REPO_BRANCH" --depth 1 "$VALUES_REPO" "$VALUES_REPO_DIR"

                                # Copy the subdirectory out (the repo contains this project under Jenkins-K8s/)
                                cp -r "$VALUES_REPO_DIR" ./aisbench_auto_tools_prefix-main
                            '''
                        }
                    }
                }
            }
        }

        stage("run prefix bench") {
            steps {
                container('aisbench') {
                    script {
                        sh """
                            echo "============================================="
                            echo "运行 Pod 名称: \${HOSTNAME}"
                            echo "宿主机节点名称: \${K8S_NODE_NAME}"
                            echo "宿主机节点 IP : \${K8S_NODE_IP}"
                            echo "模型名称: ${params.API_MODEL_NAME}"
                            echo "服务地址: ${params.SERVER_URL}"
                            echo "数据并行度: ${params.DP}"
                            echo "测试轮数: (由 TEST_CASES JSON 数组长度决定)"
                            echo "============================================="
                        """

                        // Parse SERVER_URL into --url or --host_ip/--host_port
                        def serverUrl = params.SERVER_URL.trim()
                        def urlArg = ""
                        def hostIpArg = ""
                        def hostPortArg = ""

                        if (serverUrl.startsWith("http://") || serverUrl.startsWith("https://")) {
                            // Full URL provided (Docker scenario etc.)
                            urlArg = "--url '${serverUrl}'"
                            // host_ip must still be valid IPv4 for config validation,
                            // use localhost as fallback
                            hostIpArg = "--host_ip localhost"
                            // Extract port from URL for host_port (validation only)
                            def portMatch = serverUrl =~ /:(\\d+)/
                            if (portMatch.find()) {
                                hostPortArg = "--host_port ${portMatch.group(1)}"
                            } else {
                                hostPortArg = "--host_port 8000"
                            }
                        } else {
                            // IP:PORT format
                            def parts = serverUrl.split(":")
                            hostIpArg = "--host_ip '${parts[0]}'"
                            hostPortArg = "--host_port ${parts.length > 1 ? parts[1] : '8000'}"
                        }

                        // Resolve model_path: /mnt/model/MODEL_FOLDER_NAME
                        def modelPathArg = ""
                        if (params.MODEL_FOLDER_NAME && params.MODEL_FOLDER_NAME.trim()) {
                            modelPathArg = "--model_path '/mnt/model/${params.MODEL_FOLDER_NAME.trim()}'"
                        } else {
                            modelPathArg = "--model_path '/mnt/model'"
                        }

                        // TEST_CASES → --rounds (inline JSON)
                        // Escape single quotes in TEST_CASES for shell safety
                        def testCasesEscaped = params.TEST_CASES.replaceAll("'", "'\"'\"'")

                        sh """
                            cd aisbench_auto_tools_prefix-main

                            python prefix_bench.py \
                                ${hostIpArg} \
                                ${hostPortArg} \
                                ${urlArg} \
                                --model_name '${params.API_MODEL_NAME}' \
                                ${modelPathArg} \
                                --npu_num ${params.NPU_NUM} \
                                --dp ${params.DP} \
                                --work_path /home/benchmark \
                                --dataset_path /home/dataset \
                                --output_dir ./outputs/build_${BUILD_NUMBER} \
                                --result_csv ./outputs/build_${BUILD_NUMBER}/prefix_bench_result.csv \
                                --result_jsonl ./outputs/build_${BUILD_NUMBER}/prefix_bench_result.jsonl \
                                --rounds '${testCasesEscaped}'
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "正在收集测试制品..."
                // Collect CSV, JSONL, logs, and AISBench output
                sh """
                    cd aisbench_auto_tools_prefix-main
                    mkdir -p ../${BUILD_NUMBER}

                    # Copy result files
                    if [ -d outputs/build_${BUILD_NUMBER} ]; then
                        cp -r outputs/build_${BUILD_NUMBER}/* ../${BUILD_NUMBER}/
                    fi

                    # Copy top-level result files if they exist (fallback)
                    for f in prefix_bench_result.csv prefix_bench_result.jsonl aisbench.log; do
                        if [ -f "\$f" ]; then
                            cp "\$f" ../${BUILD_NUMBER}/
                        fi
                    done

                    # Zip all outputs for archival
                    cd ../${BUILD_NUMBER}
                    zip -r results.zip . 2>/dev/null || true
                """

                archiveArtifacts artifacts: "${BUILD_NUMBER}/*.zip,${BUILD_NUMBER}/*.csv,${BUILD_NUMBER}/*.jsonl,${BUILD_NUMBER}/*.log",
                    allowEmptyArchive: true, fingerprint: true
            }
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}

def getPodYaml() {
    def imageName = params.CUSTOM_TEST_IMAGE

    return """
apiVersion: v1
kind: Pod
metadata:
  generateName: prefix-bench-${BUILD_NUMBER}
  labels:
    pipeline: prefix-bench-test
    build: "${BUILD_NUMBER}"
spec:
  nodeSelector:
    x86-app: "true"
  containers:
  - name: "aisbench"
    image: "${imageName}"
    imagePullPolicy: "IfNotPresent"
    command: [ "tail","-f","/dev/null" ]
    resources:
      requests:
        cpu: "4"
        memory: "8Gi"
      limits:
        cpu: "8"
        memory: "16Gi"
    volumeMounts:
      - mountPath: /etc/localtime
        name: timezone-volume
      - mountPath: /mnt/model
        name: models
        readOnly: true
      - mountPath: /mnt/private
        name: private
  volumes:
    - hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
        type: File
      name: timezone-volume
    - name: models
      nfs:
        path: /public_model
        server: 192.168.3.6
    - name: private
      nfs:
        path: /private
        server: 192.168.3.6
"""
}
