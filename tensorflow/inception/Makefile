
VERSION?=v3
MODEL_CKPT=$(MODEL_NAME)

# Inception model names as defined in https://github.com/tensorflow/models/blob/master/research/slim/nets/nets_factory.py
MODEL_NAME=inception_$(VERSION)

ifeq ($(VERSION),v1)
IMGSIZE=224
MODEL_CKPT_TAR=$(MODEL_CKPT)_2016_08_28.tar.gz
OUTPUT_NODE_NAME=InceptionV1/Logits/Predictions/Reshape_1
else ifeq ($(VERSION),v2)
IMGSIZE=224
MODEL_CKPT_TAR=$(MODEL_CKPT)_2016_08_28.tar.gz
OUTPUT_NODE_NAME=InceptionV2/Predictions/Reshape_1
else ifeq ($(VERSION),v3)
IMGSIZE=299
MODEL_CKPT_TAR=$(MODEL_CKPT)_2016_08_28.tar.gz
OUTPUT_NODE_NAME=InceptionV3/Predictions/Reshape_1
else ifeq ($(VERSION),v4)
IMGSIZE=299
MODEL_CKPT_TAR=$(MODEL_CKPT)_2016_09_09.tar.gz
OUTPUT_NODE_NAME=InceptionV4/Logits/Predictions
endif

YELLOW='\033[1;33m'
NOCOLOR='\033[0m'

.PHONY: all
all: profile

check_model:
ifndef TF_MODELS_PATH
	$(error TF_MODELS_PATH is not defined. Run `export TF_MODELS_PATH=path/to/your/tensorflow/models/repo`)
endif

check_tf:
ifndef TF_SRC_PATH
	$(error TF_SRC_PATH is not defined. Run `export TF_SRC_PATH=path/to/your/tensorflow/source/repo`)
endif

.PHONY: checkpoint
checkpoint:
	@echo $(YELLOW)"\nDownloading checkpoint files..."$(NOCOLOR)
	(mkdir -p model)
	(cd model; wget -nc http://download.tensorflow.org/models/$(MODEL_CKPT_TAR);)
	(cd model; tar -xvf $(MODEL_CKPT_TAR);)

.PHONY: export
export: check_model
	@echo $(YELLOW)"\nExporting GraphDef file..."$(NOCOLOR)
	(cd model; python3 $(TF_MODELS_PATH)/research/slim/export_inference_graph.py \
		--alsologtostderr \
		--model_name=$(MODEL_NAME) \
		--batch_size=1 \
		--dataset_name=imagenet \
		--image_size=$(IMGSIZE) \
		--output_file=$(MODEL_CKPT).pb;)

.PHONY: freeze
freeze: check_tf checkpoint export
	@echo $(YELLOW)"\nFreezing model for inference..."$(NOCOLOR)
	(cd model; python3 $(TF_SRC_PATH)/tensorflow/python/tools/freeze_graph.py \
		--input_graph=$(MODEL_CKPT).pb \
		--input_binary=true \
		--input_checkpoint=$(MODEL_CKPT).ckpt \
		--output_graph=$(MODEL_CKPT)_frozen.pb \
		--output_node_name=$(OUTPUT_NODE_NAME);)

.PHONY: compile
compile: freeze
	@echo $(YELLOW)"\nCompiling model to Movidius graph..."$(NOCOLOR)
	(cd model; mvNCCompile -s 12 $(MODEL_CKPT)_frozen.pb -in=input -on=$(OUTPUT_NODE_NAME);)
	@echo $(YELLOW)"Movidius graph generated! You can run inferences using ncappzoo/apps/image-classifier project.\n"$(NOCOLOR)

.PHONY: profile
profile: freeze
	@echo $(YELLOW)"\nProfiling the model..."$(NOCOLOR)
	(cd model; mvNCProfile -s 12 $(MODEL_CKPT)_frozen.pb -in=input -on=$(OUTPUT_NODE_NAME);)
	@echo $(YELLOW)"Movidius graph generated! You can run inferences using ncappzoo/apps/image-classifier project.\n"$(NOCOLOR)

.PHONY: check
check: freeze
	@echo $(YELLOW)"\nComparing results with standard Tensorflow..."$(NOCOLOR)
	(cd model; mvNCCheck -s 12 $(MODEL_CKPT)_frozen.pb -in=input -on=$(OUTPUT_NODE_NAME);)
	@echo $(YELLOW)"Movidius graph generated! You can run inferences using ncappzoo/apps/image-classifier project.\n"$(NOCOLOR)

.PHONY: run
run: compile
	@echo $(YELLOW)"\nRunning inferences using image-classifier project..."$(NOCOLOR)
	(python3 ../../apps/image-classifier/image-classifier.py --graph ./model/graph --dim $(IMGSIZE) $(IMGSIZE) --mean 127.5 --scale 0.00789 --colormode "RGB" --labels ./model/labels.txt --image ../../data/images/pic_053.jpg)

.PHONY: help
help:
	@echo $(YELLOW)"\nPossible make targets: "$(NOCOLOR);
	@echo "  make help - Shows this message.";
	@echo "  make - Builds all dependencies, but does not run this program.";
	@echo "  make checkpoint - Downloads pre-trained checkpoint files.;"
	@echo "  make export - Export the neural network model for inference.";
	@echo "  make freeze - Freeze the neural network model for inference.";
	@echo "  make compile - Convert the frozen model into Movidius graph file.";
	@echo "  make check - Compare inference results with that of TensorFlow running on CPU/GPU.";
	@echo "  make profile - Run the model on NCS and extract complexity, bandwidth and execution time for each layer.";

.PHONY: clean
clean:
	@echo $(YELLOW)"\nMaking clean..."$(NOCOLOR);
	rm -rf model
