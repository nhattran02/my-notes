# DRP-AI development

DRP-AI3 is the on-chip inference accelerator. Models are compiled offline with
the **DRP-AI TVM** toolchain and executed on the target through the DRP-AI
runtime (a TVM runtime backend).

## Toolchain setup (host)

```bash
git clone https://github.com/renesas-rz/rzv_drp-ai_tvm
cd rzv_drp-ai_tvm
# use the provided Docker image
docker build -t drp-ai_tvm ./tvm
docker run -it --rm -v $PWD:/workspace drp-ai_tvm bash
```

Inside the container set the SDK / translator paths:

```bash
export TVM_ROOT=/workspace/tvm
export TRANSLATOR=/workspace/drp-ai_translator_release
export SDK=/opt/poky/<version>          # cross SDK from Yocto populate_sdk
export PRODUCT=V2N
```

## Compile a model

```bash
python3 compile_onnx_model.py \
    ./sample/resnet18.onnx \
    -o resnet18_v2n \
    -s 1,3,224,224 \
    -i input.1
```

Output directory contains:

| File                     | Purpose                        |
| ------------------------ | ------------------------------ |
| `deploy.so`              | compiled graph (CPU + DRP-AI)  |
| `deploy.params`          | weights                        |
| `deploy.json`            | graph description              |
| `*_addrmap_intm.txt`     | DRP-AI memory map              |

## Run on target

```bash
scp -r resnet18_v2n root@<board>:/home/root/
# on the board
./tutorial_app resnet18_v2n cat.jpg
```

## Notes

- Input layout must match what was passed to `-s` / `-i` at compile time.
- Preprocessing (resize, normalize, BGR/RGB) can be offloaded to the
  **pre-processing runtime** (`preruntime`) instead of the CPU.
- Check accelerator load and timing via the runtime's profiling output
  (`DRP-AI` vs `CPU` time per layer).
- One compiled model targets one `PRODUCT`; rebuild for V2N specifically.
