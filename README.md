# EricLLM
A fast batching API to serve LLM models

Update 1/27/2024: Fixed one of the major bugs that would cause some requests to not return when under heavy load. Added extremly simple vLLM engine support. I'm trying to work around that multi-gpu bug and see if I can do it from here. Implemented the 8 bit cache but haven't tested it. Fixed the lora support. Adjusted timeout behavior.

Update 01/17/2024: Bunch of random little upgrades/bug fixes. Some speed increases. Added lora support. Successfully found a bunch of things that don't work or make things worse. I added an --embiggen flag if you want to duplicate the attention layers and make arbitrarily large frankenmodels. I've not benchmarked this against anything. The decrease in speed may not reflect properly in the console. It doesn't work well, and I suspect it's because the cache is being shared between duplicate layers and causing issues. But the API server, on the whole, works a lot better now. My use of this is expanding and I'll keep dropping the updates in.

Update 12/29/2023: I improved the way the API handles requests. It's now about 20% faster and latency is much less! Also added an average tokens/sec count per thread. You can add those together to get your tokens/sec more easily. Decreased the client request latency as well that's not reflected in the generated tokens/sec shown in the console but did reflect in my benchmarking script. There's also new gpu_balance logic primarily targeted at ~34b models ran on 2 24gb GPUs that boosted performance significantly by round-robin assigning the first 2 jobs and then splitting the third worker over the 2 GPUs to max out the CUDA cores.

## Installation

If you aren't using this inside a Text-Generation-WebUI root, you'll need to install the pre-reqs below. Otherwise, you can just drop the script in the root of TGW and run it within the cmd_windows.bat (or equivalent).

```
apt-get update
apt-get install git-lfs
git lfs install

git clone https://github.com/webbeees/EricLLM
cd EricLLM
pip install -r requirements.txt

git clone https://huggingface.co/LoneStriker/Mixtral-8x7B-Instruct-v0.1-3.0bpw-h6-exl2



python ericLLM.py --model Mixtral-8x7B-Instruct-v0.1-3.0bpw-h6-exl2 --max_prompts 3 --num_workers 5

```

## Usage

Example usage for one GPU:
```
python ericLLM.py --model ./models/NeuralHermes-2.5-Mistral-7B-5.0bpw-h6-exl2 --max_prompts 8 --num_workers 2
```
In a dual-GPU setup:
```
python ericLLM.py --model ./models/NeuralHermes-2.5-Mistral-7B-5.0bpw-h6-exl2 --gpu_split 24,24 --max_prompts 8 --num_workers 4 --gpu_balance
```
These will both launch the API with multiple workers. In the second example, performance is increased with the --gpu_balance switch that keeps the small models from splitting over GPUs. There's still work to be done on this and I think it gets CPU-bound right now when using 2 GPUs.

Test the API:

```
curl http://192.168.0.155:8000/generate -H "Content-Type: application/json" -d '{ "prompt": "This will only test single-threaded performance, but makes sure the API works because", "max_tokens": 128, "temperature": 0.7 }'
```
Note: I've been running this from WSL. Windows doesn't handle curl the same way.

If you really wanted to test out the parallel performance, you can just throw a & in between the curl requests like so:

```
curl http://192.168.0.155:8000/generate -H "Content-Type: application/json" -d '{ "prompt": "This block of code tests multi-threaded performance", "max_tokens": 128, "temperature": 0.7 }' & curl http://192.168.0.155:8000/generate -H "Content-Type: application/json" -d '{ "prompt": "This is is a second prompt that will hopefully", "max_tokens": 128, "temperature": 0.7 }' & curl http://192.168.0.155:8000/generate -H "Content-Type: application/json" -d '{ "prompt": "Here's a 3rd thing that needs to be", "max_tokens": 128, "temperature": 0.7 }'
```

You'll really need to have parallel requests like this to really see the higher speeds. The single-request speed is still good, but read the bottom section about throughput if you're looking to process a lot of tokens.

## Options

The current help menu, available via --help or -h:
```
options:
  -h, --help            show this help message and exit
  --verbose             Sets verbose
  --model MODEL_DIRECTORY
                        Sets model_directory
  --host HOST           Sets host
  --port PORT           Sets port
  --max-model-len MAX_SEQ_LEN
                        Sets max_seq_len
  --max-input-len MAX_INPUT_LEN
                        Sets max_input_len
  --gpu_split GPU_SPLIT
                        Sets array gpu_split and accepts input like 16,24
  --gpu_balance         Balance workers on GPUs to maximize throughput. Make sure --gpu_split is set to the full memory of all cards.
  --max_prompts MAX_PROMPTS
                        Max prompts to process at once
  --timeout TIMEOUT     Sets timeout
  --alpha_value ALPHA_VALUE
                        Sets alpha_value
  --embiggen EMBIGGEN
                        Duplicates some attention layers this many times to make larger frankenmodels dynamically. May increase cromulence on benchmarks.
  --compress_pos_emb COMPRESS_POS_EMB
                        Sets compress_pos_emb
  --num_experts NUM_EXPERTS
                        Number of experts in a model like Mixtral (not implemented yet)
  --cache_8bit CACHE_8BIT
                        Use 8 bit cache (not implemented)
  --num_workers NUM_WORKERS
                        Number of worker processes to use
```

## About

I’d been using vLLM to run inferencing at scale for a couple personal projects. It’s the absolute fastest thing out there for API serving. It’s awesome. But it’s got this [bug](https://github.com/vllm-project/vllm/issues/1116), or something’s wrong with my system, that’s causing me to not be able to run large models across my cards. I worked on it for days. I was so frustrated that I gave up and made my own batching API that’s feature-compatible (for what I use) in less time than I spent troubleshooting vLLM. I needed something with more performance than the text-generation-webui could provide with its single-threaded API locked by semaphore to a single request. It’s multi-threaded and rivals state of the art solutions for throughput, model loading time, and has no dependencies if you're already using the text-generation-webui.

The underly engine is ExllamaV2. You can just drop it in your Text-Generation-Webui folder and run it under that environment and be fine on dependencies. At least, it worked for me on a new one-click install. Otherwise, you’ll need to install the requirements in requirements.txt.

You can run multiple workers with the --num_workers flag. This basically duplicates everything and load balances through FastAPI. You’ll need to use this to get full utilization of a lot of memory on the smaller models.
Exllamav2 has this weird thing where the --gpu_split option is a little bugged. You want to put about half the model size (in memory) as the first GPU memory and the full memory size of the second card. So for 2x 3090’s with 24gb of memory a piece, you’d want to use something like --gpu_split 6,24 to load a 13b model evenly over the cards. At some point, they might change the way the loader works and make this bad advice. This is important to use when loading multiple workers across cards. Using --gpu_balance will round-robin distribute the workers across cards to try to maximize gpu utilization.
This is not ready for production as there’s a bunch of rough edges I need to polish up. There’s a bunch of debug outputs and the help menu isn’t finished. If you set gpu_balance, make sure gpu_split is set to the full amount of memory for 2 cards and ignore the advice about gpu_split. This is only useful for when running multiple workers and prevents the issue where splitting the model weights between cards chops the performance in half.

A lot of the cache logic came from [this exllama example](https://github.com/turboderp/exllamav2/blob/master/examples/multiple_caches.py) if you want a more focused view on exllamav2 caching, the layer slicing was [from here](https://gist.github.com/silphendio/535cd9c1821aa1290aa10d587b76a49c).

## Throughput
All tests generate 256 tokens on RTX 3090 from 32 threads on an AMD 5900x:

|       Model                                |    Speed | Workers | Max Prompts | Concurrent Requests | # 3090's |
|--------------------------------------------|----------|---------|-------------|---------------------|----------|
| NeuralHermes-2.5-Mistral-7B-8.0bpw-h8-exl2 | 238 tk/s |    2    |      16      |         16          |    1     |
| OpenOrca-Platypus2-13B-GPTQ | 185 tk/s | 2 | 8 | 16 | 1 |
| TinyLlama-1B-ckpt503-exl2 | 422 tk/s | 3 | 8 | 24 | 1 |
| goliath-120b-2.18bpw-h6-exl2 | 21 tk/s | 1 | 16 | 16 | 2 |
| turboderp_Mixtral-8x7B-instruct-exl2_5.0bpw | 41 tk/s | 1 | 16 | 16 | 2 |
| Yi-34B-200K-4.65bpw-h6-exl2 | 95 tk/s* | 3 | 16 | 16 | 2 |
| Llama2-70B-exl2-4.0bpw | 32 tk/s | 1 | 16 | 16 | 2 |

<sup>*This is with the new --gpu_balance logic to split uneven workers over GPUs. My CPU maxed out so it might go higher.</sup>

I discovered my 3090 is hitting a power limit, so I'm looking for a 12VHPWR cable that will work with my PSU. I think these numbers can go up. 

My personal benchmarking shows it about 1/3rd the speed of vLLM using the same GPU/model type. That said, that still places it as one of the fastest batching APIs available right now, and it supports the arguably superior exl2 format with variable bitrate. Loading models is much faster than vLLM, taking under 15 seconds to load a Mistral7b. I'm able to pull over 200 tokens per second from that 7b model on a single 3090 using 3 worker processes and 8 prompts per worker. Compare this to the TGW API that was doing about 60 t/s. That same benchmark was ran on vLLM and it achieved over 600 tokens per second, so it's still got the crown. The speedup on larger models is far less dramatic but still present due to the batched caching. The other significant speedup is the caching which requires multiple incoming requests to be coming in. Best performance on my system was with small models, multiple workers, prompt cache set between 8-16, and dozens of simultaneous incoming requests. At that point I became CPU-bound and wasn't able to find out what 2x 3090's can do. Since doing the round-robin assignment worked for mostly maxing out the CUDA usage, I'm confident results can be significantly improved for multiple GPUs which would allow easy horizontal scaling under a simple single API endpoint. Setting the prompt cache lower will improve latency but also drop throughput a bit. I'd love for someone to run their own benchmarks against the solution and let me know what they think!

