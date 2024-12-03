# Spam detection in Ruby

## Background

As any good Ruby developer does, I regularly review new projects by [@ankane](https://github.com/ankane). 
Most recently I came across [`transformers-ruby`](https://github.com/ankane/transformers-ruby) and have been wondering about how to leverage HuggingFace models.

A use-case finally arose in wanting to flag potential spam messages from users in a Ruby on Rails application. I initial found the [DriftingRuby episode "Detect Spam with AI"](https://www.driftingruby.com/episodes/detect-spam-with-ai) but was disatisfied with the need to use a Python microservice.

This is my exploration in how to setup and use a permissively licensed spam detection model in Ruby.

## Setup

Install [`pytorch`](https://pytorch.org/) and configure [`torch-rb`](https://github.com/ankane/torch.rb) to build using the local install:

```sh
brew install pytorch
bundle config build.torch-rb --with-torch-dir=$(brew --prefix)/Cellar/pytorch/2.5.1_1
```

Configure a lightweight Gemfile and install dependencies:

```rb
# Gemfile
source "https://rubygems.org"

gem "torch-rb"
gem "transformers-rb"
```

```sh
bundle install
```

## Example script

```rb
# spamcheck.rb
require "torch"
require "transformers"

device =
  case
  when Torch::CUDA.available?
    "cuda"
  when Torch::Backends::MPS.available?
    "mps"
  else
    "cpu"
  end

# DriftyRuby uses `mshenoda/roberta-spam`; however, `transformers-rb` does not yet support RoBERTa models.
# I found the following model which utilizes just BERT and is Apache 2.0 licensed.
model_path = "mrm8488/bert-tiny-finetuned-enron-spam-detection"

embed = Transformers.pipeline("text-classification", model: model_path, device: device)

examples = [
  "Get a free iPhone now!",
  "Hey, can we get together to watch the game tomorrow?",
  "You have won a lottery! To claim the prize, reply with your social security number.",
  "I am attaching the report for your review. Please take a look and let me know if you have any questions.",
  "BlueChew is the better way to get hard. Get your free sample today!",
]

examples.each do |example|
  puts example
  result = embed.(example)

  case result[:label]
  when "LABEL_1"
    puts "Spam (Confidence: #{result[:score]})\n\n"
  when "LABEL_0"
    puts "Ham (Confidence: #{result[:score]})\n\n"
  end
end

```

### Output

```
$ bundle exec ruby spamcheck.rb
Get a free iPhone now!
Spam (Confidence: 0.9982871413230896)

Hey, can we get together to watch the game tomorrow?
Ham (Confidence: 0.9924726486206055)

You have won a lottery! To claim the prize, reply with your social security number.
Spam (Confidence: 0.9984951019287109)

I am attaching the report for your review. Please take a look and let me know if you have any questions.
Ham (Confidence: 0.9982806444168091)

BlueChew is the better way to get hard. Get your free sample today!
Spam (Confidence: 0.9947096109390259)
```
