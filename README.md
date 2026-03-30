# RamaLama Outreachy Task 124: Documenting My Setup and Experiments

## Overview

In this task, I explored **RamaLama** and documented my setup, model downloads, and model runs. My goal was to understand how RamaLama simplifies working with AI models through a container-based workflow, and to compare different model/transport combinations in a practical way.

I started with a larger HuggingFace model, then moved to a smaller Ollama model when I ran into hardware-related issues. I also tested CPU-only execution to work around a Vulkan/device compatibility problem on my machine.

## My system

* **Operating system:** Fedora
* **Container runtime:** Podman
* **RAM:** 15 GiB
* **CPU cores:** 4
* **Swap:** 17 GiB

## 1. Installed RamaLama

I first checked that RamaLama was installed and verified the version.

### Command used

```bash
ramalama version
```

### Output

```text
ramalama version 0.17.1
```

## 2. First model: HuggingFace transport

I chose a model from HuggingFace first because I wanted to test the default workflow with a larger model and see how RamaLama handled it.

### Command used to pull the model

```bash
ramalama pull huggingface://instructlab/merlinite-7b-lab-GGUF/merlinite-7b-lab-Q4_K_M.gguf
```

### What happened

* The first download was interrupted by me while I was testing the process.
* I retried the download and let it complete successfully.
* The model size was about **4.07 GB**.

### Successful pull output

```text
100% |████████████████████████████████████████████████████████████████████████████████████████|    4.07 GB/   4.07 GB   1.51 MB/s   11m 57s
```

## 3. First run attempt with the HuggingFace model

After the model finished downloading, I tried to run it with a simple Fedora-related prompt.

### Command used

```bash
ramalama run huggingface://instructlab/merlinite-7b-lab-GGUF/merlinite-7b-lab-Q4_K_M.gguf "What are the Four Foundations of the Fedora project?"
```

### Result

The run failed with a health-check timeout.

### Error output

```text
ERROR - Failed to serve model merlinite-7b-lab-Q4_K_M.gguf, for ramalama run command
ERROR - Command 'health check of container ramalama-GzxSsJczOB' timed out after 20 seconds
```

I retried the same command, and it failed again with the same timeout.

### My observation

At first, I suspected the large model size was the problem. Later, after checking the container logs, I realized the issue was more specific: RamaLama was trying to use a Vulkan-backed path that was not compatible with my Intel HD 4000 graphics device.

## 4. Checked the container runtime

I wanted to make sure Podman itself was working correctly.

### Command used

```bash
podman run hello-world
```

### Result

Podman worked correctly, so the issue was not a broken container runtime.

### Output excerpt

```text
!... Hello Podman World ...!
```

## 5. Checked system resources

I also checked my memory and CPU availability.

### Commands used

```bash
free -h
nproc
```

### Output

```text
Mem:            15Gi       4.9Gi       5.6Gi       535Mi       5.8Gi        10Gi
Swap:           17Gi          0B        17Gi
```

```text
4
```

This showed that I had enough RAM available, so memory alone was not the main issue.

## 6. Inspected the failed RamaLama containers

I checked the container list and logs to understand the failure more clearly.

### Command used

```bash
podman ps -a
```

### What I saw

Several RamaLama containers had exited with code `139`.

### Command used to inspect logs

```bash
podman logs c259cf2e1cad
```

### Key log observation

The logs showed that the model was trying to load through Vulkan and then failing with an unsupported device error.

### Important error from the logs

```text
ggml_vulkan: device Vulkan0 does not support 16-bit storage.
llama_model_load: error loading model: Unsupported device
```

### My conclusion

The failure was not just about model size. It was also about backend/device compatibility.

## 7. Removed the failed HuggingFace model

After debugging, I removed the large model and confirmed that only the smaller Ollama model remained.

### Command used

```bash
ramalama rm huggingface://instructlab/merlinite-7b-lab-GGUF/merlinite-7b-lab-Q4_K_M.gguf
```

### Result

The model was no longer listed in `ramalama list`.

### Command used

```bash
ramalama list
```

### Output

```text
NAME                              MODIFIED       SIZE
ollama://library/tinyllama:latest 50 seconds ago 608.16 MB
```

## 8. Second model: Ollama transport

I then switched to a smaller model via the Ollama transport.

### Command used to pull the model

```bash
ramalama pull ollama://tinyllama
```

### Output

```text
Downloading ollama://library/tinyllama:latest ...
Trying to pull ollama://library/tinyllama:latest ...
Downloading tinyllama
100% |████████████████████████████████████████████████████████████████████████████████████████|  608.16 MB/ 608.16 MB   1.60 MB/s        0s
```

This model was much smaller and downloaded quickly.

## 9. First Ollama run attempt

I tried running TinyLlama directly first.

### Command used

```bash
ramalama run ollama://tinyllama "What are the Four Foundations of the Fedora project?"
```

### Result

This also failed with the same health-check timeout.

### Error output

```text
ERROR - Failed to serve model tinyllama, for ramalama run command
ERROR - Command 'health check of container ramalama-qtBP9b8dXz' timed out after 20 seconds
```

### My observation

This confirmed that the issue was not only the model size. It was also how the backend was being selected.

## 10. Forced CPU-only execution

After reviewing the logs, I realized that the machine was trying to use a GPU/Vulkan path that was not compatible with my hardware. I fixed that by forcing CPU-only execution.

### Command used

```bash
echo "What are the Four Foundations of the Fedora project?" | ramalama run --device=none --ngl 0 ollama://tinyllama
```

### Result

This time, the command succeeded.

### Output

```text
1d93583810d58dd141943aab826060abf0a1180cf3f618f2459e2ac65d3638ac
The Four Foundation(s) of the Fedora project are:

1. Fedora Core: Fedora Core is the base system of Fedora...
2. Fedora Desktop: Fedora Desktop is the graphical user interface...
3. Fedora Live: Fedora Live is a live CD/USB image...
4. Fedora Server: Fedora Server is a high-performance, lightweight server distribution...
```

### My observation

The command worked, but the answer was inaccurate. Instead of the actual Fedora foundations — **Freedom, Friends, Features, and First** — the model returned unrelated Fedora release-style concepts.

This was useful, because it showed me the difference between:

* a command that runs successfully, and
* a model that gives a correct and useful answer.

## 11. More CPU-only test prompts

I tested a few simpler prompts to confirm that TinyLlama was working consistently.

### Example command

```bash
echo "What is a noun?" | ramalama run --device=none --ngl 0 ollama://tinyllama
```

### Example output

```text
A Noun is a term used in a sentence to indicate a person, place, thing or idea...
```

The responses were coherent enough for simple prompts, but still not very reliable for accurate Fedora-specific knowledge.

I also tested a prompt about myself:

### Command used

```bash
echo "Who is Daniel Favour Dohou?" | ramalama run --device=none --ngl 0 ollama://tinyllama
```

### Output observation

The response was incorrect and unrelated to me, which reinforced that smaller models can be fast and lightweight, but they may lack contextual accuracy.

## 12. My comparison and analysis

### HuggingFace model vs Ollama model

The HuggingFace model I used was much larger and more demanding. It downloaded successfully, but it failed at runtime because RamaLama tried to use a Vulkan path that my hardware could not support.

The Ollama model was much smaller and quicker to download. However, it also failed until I forced CPU-only execution. Once I did that, it ran successfully.

### What changed when I used `--device=none --ngl 0`

This was the most important fix in my experiment.

It forced RamaLama to avoid GPU/Vulkan execution and use the CPU instead. That made the model start successfully on my machine.

### Trade-offs I observed

* **Large models** can be more capable, but they need more resources and can fail more easily on weaker hardware.
* **Smaller models** are easier to run locally, but the answers may be less accurate.
* **Transport choice** matters, but the underlying hardware and runtime backend matter even more.
* **CPU-only execution** made the system more compatible with my machine, but likely at the cost of speed and model quality.

## 13. Does this make working with AI boring?

In a good way, yes — but not completely.

RamaLama makes AI feel more boring in the sense that it hides a lot of the complicated setup behind a consistent command-line workflow. I did not have to manually build a large serving stack from scratch. I could pull a model and run it with a command.

At the same time, I still had to think carefully about:

* model size,
* transport choice,
* GPU vs CPU execution,
* and hardware compatibility.

So my conclusion is that RamaLama makes the **workflow** boring, but not the **thinking**.

## 14. What I learned

This task helped me understand that documentable AI workflows are not only about getting a model to answer a prompt. They are also about:

* choosing the right model,
* knowing your machine,
* reading logs carefully,
* and adjusting execution based on evidence.

I also learned that a successful run does not always mean a correct result. Good evaluation still matters.

## 15. Final notes

For this task, I documented:

* the RamaLama version I used,
* my model pull attempts,
* the runtime failures,
* the hardware and backend debugging process,
* my successful CPU-only run,
* and my comparison of the results.

I can now clearly say that I explored RamaLama, debugged it on my system, and learned how its container-based workflow affects AI model execution.

---

## Files and screenshots to attach

I should attach the following to my final submission:

* screenshots of each terminal step,
* any saved output files,
* and this README or blog post link, if required by the issue.

## Short final comment for the issue

I completed the task by installing RamaLama, checking the version, pulling and running a HuggingFace model, debugging the runtime failure, switching to an Ollama model, forcing CPU-only execution, and comparing the outputs and behavior of both transports.
