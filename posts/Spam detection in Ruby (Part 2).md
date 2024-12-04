# Spam detection in Ruby (Part 2)

## Converting `mshenoda/roberta-spam` to ONNX

The first step in converting `mshenoda/roberta-spam` to ONNX for usage with ankane/informers involves installing the optimum-cli:

```sh
pip install 'optimum[exporters]'
```

Unfortunately, immediately upon invocation it threw an error:
```sh
~ ❯ optimum-cli
Traceback (most recent call last):
  File "/Users/jonahgeorge/.pyenv/versions/3.10.8/bin/optimum-cli", line 5, in <module>
    from optimum.commands.optimum_cli import main
  File "/Users/jonahgeorge/.pyenv/versions/3.10.8/lib/python3.10/site-packages/optimum/commands/__init__.py", line 16, in <module>
    from .env import EnvironmentCommand
  File "/Users/jonahgeorge/.pyenv/versions/3.10.8/lib/python3.10/site-packages/optimum/commands/env.py", line 18, in <module>
    from transformers import __version__ as transformers_version
  File "/Users/jonahgeorge/.pyenv/versions/3.10.8/lib/python3.10/site-packages/transformers/__init__.py", line 26, in <module>
    from . import dependency_versions_check
  File "/Users/jonahgeorge/.pyenv/versions/3.10.8/lib/python3.10/site-packages/transformers/dependency_versions_check.py", line 16, in <module>
    from .utils.versions import require_version, require_version_core
  File "/Users/jonahgeorge/.pyenv/versions/3.10.8/lib/python3.10/site-packages/transformers/utils/__init__.py", line 27, in <module>
    from .chat_template_utils import DocstringParsingException, TypeHintParsingException, get_json_schema
  File "/Users/jonahgeorge/.pyenv/versions/3.10.8/lib/python3.10/site-packages/transformers/utils/chat_template_utils.py", line 36, in <module>
    from PIL.Image import Image
  File "/Users/jonahgeorge/.pyenv/versions/3.10.8/lib/python3.10/site-packages/PIL/Image.py", line 100, in <module>
    from . import _imaging as core
ImportError: dlopen(/Users/jonahgeorge/.pyenv/versions/3.10.8/lib/python3.10/site-packages/PIL/_imaging.cpython-310-darwin.so, 0x0002): Library not loaded: /opt/homebrew/opt/libimagequant/lib/libimagequant.0.0.dylib
  Referenced from: <EE2C4FB1-A155-382A-9BBE-16AE7D8F261B> /Users/jonahgeorge/.pyenv/versions/3.10.8/lib/python3.10/site-packages/PIL/_imaging.cpython-310-darwin.so
  Reason: tried: '/opt/homebrew/opt/libimagequant/lib/libimagequant.0.0.dylib' (no such file), '/System/Volumes/Preboot/Cryptexes/OS/opt/homebrew/opt/libimagequant/lib/libimagequant.0.0.dylib' (no such file), '/opt/homebrew/opt/libimagequant/lib/libimagequant.0.0.dylib' (no such file), '/usr/local/lib/libimagequant.0.0.dylib' (no such file), '/usr/lib/libimagequant.0.0.dylib' (no such file, not in dyld cache), '/opt/homebrew/Cellar/libimagequant/4.3.3/lib/libimagequant.0.0.dylib' (no such file), '/System/Volumes/Preboot/Cryptexes/OS/opt/homebrew/Cellar/libimagequant/4.3.3/lib/libimagequant.0.0.dylib' (no such file), '/opt/homebrew/Cellar/libimagequant/4.3.3/lib/libimagequant.0.0.dylib' (no such file), '/usr/local/lib/libimagequant.0.0.dylib' (no such file), '/usr/lib/libimagequant.0.0.dylib' (no such file, not in dyld cache)
```

Tracing down this line, it appears that `PIL` comes from the `Pillow` dependency and the version I previously had installed was somehow broken.

I gave it a hail-mary upgrade:

```sh
~ ❯ pip install --upgrade pillow
Requirement already satisfied: pillow in ./.pyenv/versions/3.10.8/lib/python3.10/site-packages (9.3.0)
Collecting pillow
  Downloading pillow-11.0.0-cp310-cp310-macosx_11_0_arm64.whl.metadata (9.1 kB)
Downloading pillow-11.0.0-cp310-cp310-macosx_11_0_arm64.whl (3.0 MB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 3.0/3.0 MB 13.1 MB/s eta 0:00:00
Installing collected packages: pillow
  Attempting uninstall: pillow
    Found existing installation: Pillow 9.3.0
    Uninstalling Pillow-9.3.0:
      Successfully uninstalled Pillow-9.3.0
Successfully installed pillow-11.0.0
```

```sh
~ ❯ optimum-cli                            
usage: optimum-cli

positional arguments:
  {export,env,onnxruntime}
    export              Export PyTorch and TensorFlow models to several format.
    env                 Get information about the environment used.
    onnxruntime         ONNX Runtime optimize and quantize utilities.

options:
  -h, --help            show this help message and exit
```

Next step was attempting the actual model conversion, which ran without errors:

```sh
optimum-cli export onnx --model mshenoda/roberta-spam mshenoda_roberta_spam_onnx/
```

## Example script
```rb
source "https://rubygems.org"

gem "torch-rb"
gem "informers"
```

```rb
# spamcheck.rb
require "torch"
require "informers"

# https://huggingface.co/mshenoda/roberta-spam
model = Informers.pipeline(
  "text-classification",
  "mshenoda/roberta-spam",
  model_file_name: "../model",
  quantized: false
)

sentences = [
  "Get a free iPhone now!",
  "Hey, can we get together to watch the game tomorrow?",
  "You have won a lottery! To claim the prize, reply with your social security number.",
  "I am attaching the report for your review. Please take a look and let me know if you have any questions.",
  "BlueChew is the better way to get hard. Get your free sample today!",
]

sentences.each do |example|
  outputs = model.(example)
  puts outputs
end
```



Moving model to cache dir to prevent HuggingFace Hub lookup:
```
mkdir -p ~/.cache/informers/mshenoda
mv mshenoda_roberta_spam_onnx ~/.cache/informers/mshenoda/robert-spam
```


```
bundle exec ruby spamcheck.rb
/Users/jonahgeorge/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/informers-1.2.0/lib/informers/models.rb:64:in `from_pretrained': Unsupported model type: roberta (Informers::Error)
	from /Users/jonahgeorge/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/informers-1.2.0/lib/informers/pipelines.rb:1445:in `block in load_items'
	from /Users/jonahgeorge/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/informers-1.2.0/lib/informers/pipelines.rb:1431:in `each'
	from /Users/jonahgeorge/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/informers-1.2.0/lib/informers/pipelines.rb:1431:in `load_items'
	from /Users/jonahgeorge/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/informers-1.2.0/lib/informers/pipelines.rb:1408:in `pipeline'
	from spamcheck.rb:6:in `<main>'
```
