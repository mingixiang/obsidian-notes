
启动vllm容器
docker run -itd --name vllm-8b-server \
    --privileged \
    --network=host \
    --ipc=host \
    --shm-size=16g \
    --device=/dev/davinci0 \
    --device=/dev/davinci1 \
    --device=/dev/davinci2 \
    --device=/dev/davinci3 \
    --device=/dev/davinci4 \
    --device=/dev/davinci5 --device=/dev/davinci6 --device=/dev/davinci7 \
    --device=/dev/davinci_manager --device=/dev/hisi_hdc \
    -v /usr/local/dcmi:/usr/local/dcmi \
    -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi \
    -v /usr/local/npu/driver/lib64/:/usr/local/npu/driver/lib64/ \
    -v /usr/local/npu/version.info:/usr/local/npu/version.info \
    -v /etc/ascend_install.info:/etc/ascend_install.info \
	-v /data/8b/qwen3-8b-experiment-merged:/workspace/qwen3-8b-experiment-merged \
    -v /root/.cache:/root/.cache \
    --entrypoint=bash \
    quay.io/ascend/vllm-ascend:v0.23.0rc1

在容器中启动模型
export LD_LIBRARY_PATH=/usr/local/npu/driver/lib64/driver:/usr/local/npu/driver/lib64:$LD_LIBRARY_PATH
export HCCL_NPU_SOCKET_PORT_RANGE=16667-17777
export ASCEND_RT_VISIBLE_DEVICES=2   # 让 vLLM 看见指定NPU
启动模型
nohup vllm serve /workspace/qwen3-8b-experiment-merged \#指定模型
  --host 0.0.0.0 \#指定启动IP
  --port 8001 \#指定启动端口
  --tensor-parallel-size 1 \#指定NPU数量，张量并行
  --served-model-name qwen3-8b-experiment \#模型命名
  --trust-remote-code \
  --dtype bfloat16 \#模型权重与数值精度 config.json的torch_dtype字段会显示
  --max-model-len 32768 \#上下文长度，token 数
  --max-num-seqs 32 \#最大请求次数
  --gpu-memory-utilization 0.90 \#NPU使用比例——直接影响KV Cache影响并发
  --language-model-only \#只加载语言模型部分
  --async-scheduling \#异步调度——改善 TPOT 和吞吐
  --compilation-config '{"mode": 3, "cudagraph_mode": "FULL", ![[Pasted image 20260916213042.png]]
  "cudagraph_capture_sizes": [1,2,4,8,16,32]}' \#预先为哪些 batch size捕获计算图> server_experiment.log 2>&1 &>#重定向日志到server.log