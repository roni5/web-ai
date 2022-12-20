# In-browser AI

- ONNX Runtime
- Hugging Face compatible

## Status

The library is under active development. If something does not work correctly, please file an issue on GitHub. Contributions are very welcome.

## Model types

### Text models

- Sequence-to-sequence (`TextModelType.Seq2Seq`). These models are used to transform the text into another text. Examples of such transformations are translation, summarization, and grammar correction.
- Feature extarction (`TextModelType.FeatureExtraction`). These models are used to transform the text into an array of numbers - embedding. Generated vectors are useful for semantic search or cluster analysis because embeddings of semantically similar text are similar and can be compared using [cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity).

### Image models

- Semantic segmentation (`ImageModelType.Segmentation`). These models cluster images into parts which belong to the same object class. In other words, segmentation models detect exact shape of the objects in the image and classify them. The example of image segmentation is below.

  ![Semantic segmentation example](/images/segment.jpg)

- Object detection (`ImageModelType.ObjectDetection`). These models find objects in the images, classify them, and generate bounding boxes for the objects. The example of the object detection is below.

  ![Object detection example](/images/detection.jpg)

- Classification (`ImageModelType.Classification`). These models do not find exact objects in the images but they only determine what type of object is the most likely in the image. Because of that, this type of models is the most useful when there is only one distinct class of objects present in the image. In the example below, the image is classified as "Egyptian cat".

  ![Object detection example](/images/classification.jpg)

## Installation

The library can be insatlled via `npm`:

```
npm install in-browser-ai
```

## Create model instance

### Create model from ID

The first way of creating a model is using the model identifier. This method works only for the built-in models.

For text models:

```TypeScript
import { TextModel } from "in-browser-ai";

const result = TextModel.create("grammar-t5-efficient-tiny")
console.log(result.elapsed)
const model = result.model
```

For image models:

```TypeScript
import { ImageModel } from "in-browser-ai";

const model = ImageModel.create("yolos-tiny-quant")
console.log(result.elapsed)
const model = result.model
```

### Create model from metadata

The second way to create a model is via the model metadata. This method allows to use custom ONNX models. In this case, we need
to use a specific model class. Please note that when creating the model from the metadata, you need to call an `init()` method before using the model. This is needed to create inference sessions, download configurations files, and create internal structures.

#### Text models

The metadata for text models is defined by the `TextMetadata` class. Not all fields are required for the model creation. The minimal example for the `Seq2Seq` model is:

```TypeScript
import { Seq2SeqModel, TextMetadata } from "in-browser-ai";

const metadata: TextMetadata = {
    modelPaths: new Map<string, string>([
      [
        "encoder",
        "https://huggingface.co/visheratin/t5-efficient-tiny-grammar-correction/resolve/main/encoder_model.onnx",
      ],
      [
        "decoder",
        "https://huggingface.co/visheratin/t5-efficient-tiny-grammar-correction/resolve/main/decoder_with_past_model.onnx",
      ],
    ]),
    tokenizerPath: "https://huggingface.co/visheratin/t5-efficient-tiny-grammar-correction/resolve/main/tokenizer.json",
  }

const model = new Seq2SeqModel(metadata);
const elapsed = await model.init();
console.log(elapsed);
```

The minimal example for the `FeatureExtraction` model is:

```TypeScript
import { FeatureExtractionModel, TextMetadata } from "in-browser-ai";

const metadata: TextMetadata = {
    modelPaths: new Map<string, string>([
      [
        "encoder",
        "https://huggingface.co/visheratin/t5-efficient-tiny-grammar-correction/resolve/main/encoder_model.onnx",
      ],
    ]),
    tokenizerPath: "https://huggingface.co/visheratin/t5-efficient-tiny-grammar-correction/resolve/main/tokenizer.json",
  }

const model = new FeatureExtractionModel(metadata);
const elapsed = await model.init();
console.log(elapsed);
```

#### Image models

The metadata for image models is defined by the `ImageMetadata` class. Not all fields are required for the model creation. The minimal example for all image models is:

```TypeScript
import { ImageMetadata } from "in-browser-ai";

const metadata: ImageMetadata = {
    modelPath: "https://huggingface.co/visheratin/segformer-b0-finetuned-ade-512-512/resolve/main/b0.onnx.gz",
    configPath: "https://huggingface.co/visheratin/segformer-b0-finetuned-ade-512-512/resolve/main/config.json",
    preprocessorPath: "https://huggingface.co/visheratin/segformer-b0-finetuned-ade-512-512/resolve/main/preprocessor_config.json",
  }
```

Then, the model can be created:

```TypeScript
import { ClassificationModel, ObjectDetectionModel, SegmentationModel } from "in-browser-ai";

const model = new ClassificationModel(metadata);
// or
const model = new ObjectDetectionModel(metadata);
// or
const model = new SegmentationModel(metadata);

const elapsed = await model.init();
console.log(elapsed);
```

## Data processing

### Text models

The processing is done using a `process()` method.

`Seq2Seq` models output text:

```TypeScript
const input = "Test text input"
const output = await model.process(input)
console.log(output.text)
console.log(`Sentence of length ${input.length} (${output.tokensNum} tokens) was processed in ${output.elapsed} seconds`)
```

`FeatureExtraction` models output numeric array:

```TypeScript
const input = "Test text input"
const output = await model.process(input)
console.log(output.result)
console.log(`Sentence of length ${input.length} (${output.tokensNum} tokens) was processed in ${output.elapsed} seconds`)
```

### Image models

For the image models, the processing is also done using a `process()` method.

`Segmentation` models output HTML canvas, which can be overlayed on the original image:

```TypeScript
const input = "https://upload.wikimedia.org/wikipedia/commons/thumb/8/81/Georgia5and120loop.jpg/640px-Georgia5and120loop.jpg"
const output = await model.process(input)
var destCtx = canvas.getContext("2d");
destCtx.globalAlpha = 0.4;
destCtx.drawImage(result.canvas, 0, 0, result.canvas.width, result.canvas.height,
  0, 0, canvas.width, canvas.height);
console.log(output.elapsed)
```

If you want to determine the class from the output canvas, you can use the `getClass()` method:

```TypeScript
// xCoord and yCoord are coordinates of the target pixel on the canvas
const rect = canvas.getBoundingClientRect();
const ctx = canvas.getContext("2d");
const x = xCoord - rect.left;
const y = yCoord - rect.top;
const c = ctx!.getImageData(x, y, 1, 1).data;
const className = model.instance.getClass(c);
console.log(className);
```

`Object detection` models output a list of bounding box predictions along with their classes and colors. Bounding boxes can be used to draw them over the original image:

```TypeScript
const input = "https://upload.wikimedia.org/wikipedia/commons/thumb/8/81/Georgia5and120loop.jpg/640px-Georgia5and120loop.jpg"
const output = await model.process(input)
for (let object of output.objects) {
  var rect = document.createElementNS("http://www.w3.org/2000/svg", "rect");
  rect.setAttributeNS(null, "x", (sizes[0] * object.x).toString());
  rect.setAttributeNS(null, "y", (sizes[1] * object.y).toString());
  rect.setAttributeNS(null, "width", (sizes[0] * object.width).toString());
  rect.setAttributeNS(
    null,
    "height",
    (sizes[1] * object.height).toString()
  );
  const color = object.color;
  rect.setAttributeNS(null, "fill", color);
  rect.setAttributeNS(null, "stroke", color);
  rect.setAttributeNS(null, "stroke-width", "2");
  rect.setAttributeNS(null, "fill-opacity", "0.35");
  // svgRoot is a root SVG element on the page
  svgRoot.appendChild(rect);
}
```

`Classification` models output an array of predicted classes along with the confidence scores in range [0,1] sorted by confidence in the descending order. When running the `process()` method, you can specify the number of returned predictions (default is 3):

```TypeScript
const input = "https://upload.wikimedia.org/wikipedia/commons/thumb/8/81/Georgia5and120loop.jpg/640px-Georgia5and120loop.jpg"
const output = await model.process(input, 5)
for (let item of output.results) {
  console.log(item.class, item.confidence)
}
```

## Built-in models

### Image models

#### Semantic segmentation

#### Classification

#### Object detection

### Text models

#### Grammar correction

#### Text feature extraction

## Future development

- Improve grammar correction model.
- Extend text models beyond T5.
- Distil [Flan-T5-small](https://huggingface.co/google/flan-t5-small) model to make it more usable in the browser.
- Add audio models ([Whisper-small](https://huggingface.co/openai/whisper-small)).