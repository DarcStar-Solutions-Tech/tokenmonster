## Training Vocabularies

There are 4 steps to generating a vocabulary.

### 1. Prepare your dataset

To train a vocabulary, you need a dataset as a single plain text file. This dataset should represent exactly what you want to vocabulary to represent, and in the same proportions. For example, if you want the vocabulary to cover both English and French then you should ensure the dataset is 50% English and 50% French. Around 1GB is a reasonable dataset size.

### 2. Generate tokens

The next stage is to use the tool `getalltokens` to filter and export a list of all tokens that appear in the dataset. This is the stage at which you choose your charset and optimization mode settings. It takes around 30 minutes.

### 3. Train vocabulary

Using `trainvocab`, the tokens list is distilled down to your target vocabulary size. This takes 12 - 24 hours.

### 4. Export vocabulary

`exportvocab` then converts the output of `trainvocab` into a finished TokenMonster vocabulary file that can be used with the Python, Go & Javascript tokenizers. You can add special tokens and resize the vocabulary.

Let's go through each stage in more detail.

## First compile or download the binaries

You need the `getalltokens`, `trainvocab` and `exportvocab` binaries. You can either download these prebuilt for your architecture from [here](https://huggingface.co/alasdairforsythe/tokenmonster/tree/main/binaries). Or you can build them yourself from source using Go 1.20 as follows:

### Linux:

Install Golang:
```
apt install golang
```
Clone TokenMonster repository:
```
git clone github.com/alasdairforsythe/tokenmonster
cd tokenmonster/training
```
Build the binaries:
```
go mod init tokenmonster
go mod tidy
go build getalltokens.go
go build trainvocab.go
go build exporttokens.go
```

### Windows

Download the Windows binaries directly from [here](https://huggingface.co/alasdairforsythe/tokenmonster/tree/main/binaries/windows_x86_64).
To use them, launch `Windows Powershell` and navigate to the directory where the executables are. Then type `./getalltokens`. It should display the usage instructions, if not you probably need to add execute permissions (Google and ChatGPT both know how to do that.)

## Prepare your dataset

The vocabulary is generated from the words, phrases and grammar in the dataset. It's important to prepare a dataset that properly represents what you want to tokenize. If a word is not in your dataset, it won't be in your vocabulary. Additionally, the importance of a word within the vocabulary is directly related to the frequency that it appears within the dataset. It's important therefore to ensure that different styles of writing, different languages, etc. that you intend to be included, make up a reasonable portion of the dataset. It is not necessary to distribute styles of text throughout the dataset; it's okay, for example, to have the first half of the file English, and the second half French.

I provide the datasets I used to generate the prebuilt vocabularies on [Hugging Face](https://huggingface.co/datasets/alasdairforsythe/text-english-code-fiction-nonfiction). You're welcome to use any of these to complement your dataset. Two helper Python scripts are included `download_code_samples.py` and `extract_text_from_jsonl_parquet.py` which I used to generate those datasets.

The diversity and range of the dataset depends upon whether it's intended use is specialized or general. Let's say, for example, that it's for a text-generation model that writes short stories in the style of Dr. Seuss when prompted with a subject. If that were the case, your dataset for the vocabulary can and should be the same dataset as used for training the model. In this case, you don't need to be worried about overfitting on the vocabulary because the output is always in the same style as the training data. On the other hand, if you are generating a vocabulary for a ChatGPT-like general model, you would want to be much more careful to avoid overfitting and likely use a different dataset to that used for training the model.

## Generate tokens

Once you have your dataset ready as a single .txt file you can generate the tokens using `getalltokens`. This process takes from a minute to an hour, depending on the dataset size and how many threads you give it. For the pre-built vocabularies, `mode=clean` took around 20 minutes, and `mode=strict` took 2 minutes.

It's important to choose the correct settings, so I will go through them one by one.

```
Usage of ./getalltokens:
  -charset string
        one of: UTF-8, UTF-16, none (required)
  -chunk-size int
        the number of bytes processed at a time, higher is faster but requires more RAM (default 100MB)
  -dataset string
        filename of the dataset plain-text (required)
  -disable-capcode
        disables capcode normalizations (default false)
  -max-token-length int
        the maximum length of a token (default 40)
  -micro-chunks int
        the higher this number, the slower it is but it will reduce peak memory usage (default 5)
  -min-occur int
        tokens will be trimmed if they occur less frequently than this in the dataset (default 1 per 10MB)
  -min-occur-byte int
        single bytes will be trimmed if they occur less frequently than this in the dataset (default min-occur)
  -min-occur-chunk int
        tokens will be trimmed if they occur less frequently than this per chunk (default 4)
  -min-occur-micro-chunk int
        tokens will be trimmed if they occur less frequently than this per micro-chunk (default 2)
  -mode string
        0 = unfiltered, 1 = clean, 2 = balanced, 3 = consistent, 4 = strict (required)
  -output string
        output filename for the dictionary (required)
  -workers int
        number of worker threads to run (default 8)
```

### -charset

Whilst this is called `-charset` it mostly affects the normalization. In most cases you should choose `charset=utf8`. `UTF-16` is experimental and does not support capcode.

On the other hand, if you are tokenizing something that is not a language and does not use words, for example a gene sequence, choose `-charset=none`. Use `None` whenever you don't want spacing or characters to be normalized.

### -disable-capcode

If you choose `charset=utf8` this by default enables capcode, which is an alternative encoding for upper-casing. To disable capcode pass the `-disable-capcode` flag. You would want to disable capcode if your target language does not use capital letters, or if it's a code-only model. However, even with capcode disabled, `-charset UTF-8` will still be normalized for accents, word beginnings, number beginnings and use a forward delete token. To completely disable all normalization, use `-charset none`. 

### -mode

The optimization `mode` is one of the most important parameters, as this completely changes the way your vocabulary works.

`-mode=unfiltered` as described does not filter any tokens. Use this when your dataset is already clean, such as a text file that contains only gene sequences. Do not use this for natural language, and especially do not use it for code because it will result in overfitting.

`-mode=clean` provides minimal normalization to avoid overfitting. Specifically, words are forced to begin with a space intead of end with one, and it restricts the amount of whitespace that can appear in a token alongside letters or numbers.

`-mode=balanced` optimizes tokens for whole words. A token is restricted from covering one and a half words, or ending on a capcode modifier, with exceptions.

`-mode=consistent` at this level there are significant restrictions on combining letters, numbers, delimiters & punctuation within the same token. Some combinations are allowed, for example a word can end with a comma or a space, but in most cases these are now forced into separate tokens to enforce consistency. Unlike `clean` & `balanced`, where there can be multiple versions of a single word, such as ` then`, ` then - `, ` "then`, ` then.`, in `-mode=consistent` there will be a handful of basic variations at most. There are also restrictions on open-closers such as `([{'"` from being combined with words or numbers.

`-mode=strict` attempts to have only 1 token for each word, however it is written. `HELLO`, `"Hello"` & `hello!` will all be tokenized with the same ` hello` token, combined with capcode and punctuation tokens. It is allowed for tokens to cover multiple words, so ` how` and ` how are you` may be separate tokens. Open-closers such as `([{'"` are restricted from being combined with other marks, with some exceptions.

As a rule of thumb, small models should use `strict` or `consistent`, medium models should use `consistent` or `balanced`, and large models should use `balanced` or `clean`. You can view the difference between them on the [online viewer](https://bot.co/tokenmonster/).

### -max-token-length

The default and maximum is 40, I suggest leaving it at that, unless you have a good reason to limit it. Reducing max-token-length is unlikely to make the training or tokenization faster because the algorithm is optimized for `-max-token-length 40`.

### -min-occur

Tokens that occur less frequently than this throughout the dataset will be deleted at the end. The default value makes it `10` if the dataset is 100MB, and `100` if the dataset is 1GB.

### -chunk-size, -micro-chunks, -min-occur-chunk, -min-occur-micro-chunk

When running `getalltokens` there is a tradeoff between speed, RAM and the pruning rate. To manage this, we have these parameters.

`-chunk-size` is the number of bytes that are processed before the tokens are pruned for minimum frequency of `min-occur-chunk`. Then within each chunk, the tokens are sorted and pruned  `-micro-chunks` number of times, each time with tokens occurring less often than `-min-occur-micro-chunk` being deleted.

The default settings of `-chunk-size 100MB -micro-chunks 5` is unlikely to use more than 8 - 16 GB of RAM, depending on the optimization mode (lower optimization mode uses more RAM). If you find yourself swapping, a little is okay, but if it's too much then kill the process and run it again with `-micro-chunks 10`.

If `-micro-chunks 10` is still using too much RAM, you can use `-chunk-size 10MB -min-occur-chunk 2 -micro-chunks 10 -min-occur-micro-chunk 1` which should use very little RAM.

### -min-occur-byte

This is how many times an individual byte must occur before it gets given a token. By default this is the same as `min-occur`, but if you know your dataset is clean you might want to set it to `1` so that all individual bytes that occur will have a token that covers them. Or you might want to set it to `3` so that individual bytes that can occur have tokens, but a little bit of corrupt data in the dataset won't result in tokens being allocated.

Byte tokens are different to regular tokens in that they're not filtered out of the vocabulary. Single byte tokens you count here will most likely end up in your final vocabulary, although you can specifically exclude them at the next stage.

### -workers

This is the number of threads used. More is faster, give it as many as you have available.

## Train vocabulary

This is the main stage of vocabulary generation. Unlike `getalltokens` this process uses very little RAM, but it does take a long time. On 8 threads, it'll take between 12-24 hours to generate a final vocabulary. There is a `-fast` option that will produce a lower-quality vocabulary in about an hour, which is intended for testing the viability of a vocabulary before doing the full training.

```
Usage of ./trainvocab:
  -dataset string
        filename of the dataset plain-text (required)
  -dictionary string
        filename of the dictionary generated by getalltokens or any of the saved output files from this app (required)
  -dictionary2 string
        a second dictionary that will be merged with the first (optional)
  -dir string
        directory to save the results within (required)
  -exclude-other-bytes
        any single bytes not specifically included will not receive tokens, even if they were in the training dataset (default false)
  -fast
        runs 10x faster but the vocabulary might not be as optimal (default false)
  -include-128-bytes
        include tokens representing every ASCII character inc. control characters (default false)
  -include-256-bytes
        include tokens representing every possible byte (default false)
  -include-ascii-bytes
        include tokens for every printable ASCII character, inc. \r\n\t (default false)
  -include-missing-bytes
        add tokens for any single bytes found in the dataset that are not tokens already (default false)
  -include-utf8-bytes
        include tokens for every byte that can occur in UTF-8 text (default false)
  -keep-trying int
        program will exit when unable to find a better match this many times in a row (default 1000)
  -midway-target int
        beneath this the full dataset is used for every worker (default 6x vocab-size)
  -percentage int
        percentage of the dataset given to each worker before midway-target (default 15)
  -special string
        filename of a JSON file containing special tokens (optional)
  -vocab-size int
        vocabulary size, e.g. 32000 (required)
  -workers int
        number of worker threads to run, excluding main thread
```

`-dataset` is the dataset. `-dictionary` is the tokens file from `getalltokens`. `-dir` is the output directory. `-vocab-size` is your target vocabulary size. `-workers` is the number of threads to run (best to set it to 1 less than the number of CPU threads you have.)

Before running `trainvocab` you first need to decide on the parameters for what to do with single-byte tokens. Those are tokens for individual bytes, such standard English characters that are represented with ASCII, and starting or continuation bytes of multi-byte sequences in UTF-8.

Unlike regular tokens, all single-byte tokens are included in every vocabulary. In total there are 256 possible single-byte tokens. So for large vocabularies it's common to include all of them. This ensures that any data can be tokenized with the vocabulary, even binary data. However, that's a bit wasteful for smaller vocabularies, especially for specialized models where you know that some of those bytes will never be used by the model.

`-include-256-bytes` reserves 256 tokens for all possible single bytes.

`-include-128-bytes` includes the full set of 128 ASCII character codes, including control characters.

`-include-ascii-bytes` includes all printable ASCII characters, as well as `\r\n\t` (newlines and tabs), but none of the control characters. This is what you want for small models that are only trained on English text.

`-include-utf8-bytes` gives a token for every byte that could make up any part of a valid UTF-8 character sequence.

`-include-missing-bytes` dynamically adds tokens for all bytes discovered during training that don't already have tokens.

`-exclude-other-bytes` instructs `trainvocab` to ignore single-byte token from the tokens file generated by `getalltokens`.

For the prebuilt vocabularies, I mostly used `-include-256-bytes` for 50256, 65536 & 100256 sizes. I used `-include-utf8-bytes -exclude-other-bytes` for most of the others, and 8000 and less vocabulary size I used `-include-ascii-bytes -exclude-other-bytes`.

When `trainvocab` has finished, there will be hundreds of files in the `-dir` output directory. The one you want is the alphabetically first file, as this is the list of vocab-size tokens that tokenized the dataset with the least number of tokens. Once you've exported this as a vocabulary, you can delete this directory. The other files that begin with numbers are all the other vocabularies that were within 1% of the best vocabulary, and the other files are intermediary files that can be used to resume the training from any of those intermediary points.

Note that the file format of the tokens files in the output directory is the same format as the tokens generated by `getalltokens`. As such, you can use any of the output files from `trainvocab` as the input `-dictionary` for `trainvocab`. For example, you could use the final tokens from a 50,000 size vocabulary as the input dictionary for training a 10,000 size vocabulary. Doing so is not recommended because the optimal 10,000 tokens is not necessarily a subset of the optimal 50,000 tokens, but if you're in a hurry, you can do it.

### -special

This parameter is used to add "special tokens". A special token is special because (1) it's always included in the vocabulary, and (2) regular tokens cannot contain it. Special tokens can be added during training, or at a later time.

To include special tokens during training, `-special` refers to a JSON file in the following format:
```json
{ "special": [ "TOKEN1", "TOKEN2", "TOKEN3" ] }
```

## Export vocabulary

Once a vocabulary has been generated it's not yet in the vocabulary format, but is still just a list of tokens. To convert it to a vocabulary for use with the tokenizing libraries, you use `exportvocab`.

The standard usage is straightforward:
```
./exportvocab -input directory -output file.vocab
```
This will automatically select the best vocabulary from `directory` (as generated by `trainvocab`) and output the final vocabulary to `file.vocab`.

That's finished. However, there are other things you can do with `exportvocab`:

```
Usage of ./exportvocab:
  -add-single-bytes string
        enter "256", "128", "ascii" or "utf8" to add tokens for those individual bytes (optional)
  -add-special-token string
        a single special token to add to the vocabulary (optional)
  -delete-single-bytes
        deletes all the single byte tokens except those specified from add-single-bytes (optional)
  -delimiter string
        delimiter to use between each token for output-txt (optional) (default "\n")
  -exists string
        check if a token exists in the vocabulary (optional)
  -input string
        tokens file or directory from trainvocab, if directory it will load the best performing tokens file in the directory (optional)
  -input-json string
        filename of a JSON file containing tokens to add or delete, format: {"add":["ab","cd"],"special":["</eos>"],"delete":["cheese"]} (optional)
  -input-vocab string
        an existing TokenMonster vocabulary file (optional)
  -order-by-score
        orders output-txt by token score (descending) instead of alphabetically (optional) (default false)
  -output string
        filename of the vocabulary to output (optional)
  -output-tokens string
        converts a vocabulary back to a tokens file that can be used with trainvocab (optional)
  -output-txt string
        filename to export tokens in a text file for curiosity (optional)
  -resize int
        resizes the vocabulary to this many tokens by deleting the worst scoring tokens (optional)
  -unk string
        set to true or false to enable or disable the UNK token (optional)
```

You can input a tokens file with `-input` or an existing vocabulary file with `-input-vocab`. You can then make make various modifications and save it as either a vocabulary with `-output`, or back to a tokens file with `-output-tokens` (which can be used for training again), or as a text file with `-output-txt`.

`-add-single-bytes` & `-add-special-token` allow you to add single-byte or special tokens. If you combine this with `-resize` you can also keep the vocabulary within a target size. For example, to add a special token to an existing vocabulary of size 10000, but not increase the vocabulary size, you could use:
```
./exportvocab -input-vocab myvocab.vocab -add-special-token "<eos>" -resize 10000 -output mynewvocab.vocab
```

`-unk` can be used to enable or disable the UNK token. If enabled, during tokenization, any byte for which there is no token will be covered with the UNK token. If disabled, a byte without a token is skipped. Vocabularies that used `-include-256-bytes` cannot have an UNK token because all bytes already have tokens.

`-exists` can be used if you want to check whether a token exists in a vocabulary. You should note that words generally begin with a space. Example:
```
./exportvocab -input-vocab myvocab.vocab -exists " cheesecake"
```