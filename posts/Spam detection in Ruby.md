# Spam detection in Ruby

```rb
source "https://rubygems.org"

gem "torch-rb"
gem "transformers-rb"
```

```rb
require "transformers"

embed = Transformers.pipeline(
  "text-classification",
  model: "mrm8488/bert-tiny-finetuned-enron-spam-detection",
  device: "cpu"
)

examples = [
  "Get a free iPhone now!",
  "Hey, can we get together to watch the game tomorrow?",
  "You have won a lottery! To claim the prize, reply with your social security number.",
  "I am attaching the report for your review. Please take a look and let me know if you have any questions.",
  "Blue chew is the better way to get hard. Get your free sample today!",
]

examples.each do |example|
  puts example
  result = embed.(example)

  case result[:label]
  when "LABEL_1"
    puts "Spam (Confidence: #{result[:score]})"
  when "LABEL_0"
    puts "Ham (Confidence: #{result[:score]})"
  end

  puts ""
end
```

Output:
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

Blue chew is the better way to get hard. Get your free sample today!
Spam (Confidence: 0.995394766330719)
```
