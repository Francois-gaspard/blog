---
title: IO Streaming
description: Reducing memory usage, improving performance with IO streaming.
date: 2024-06-21 13:39
---

# IO and Streaming in Ruby

## What are streams?

Ruby streams allow **sequential data processing in chunks**, rather than loading everything into memory at once. 
They use IO objects to read or write data incrementally, reducing memory usage and improving performance for large datasets. 
This is ideal for tasks like processing large files or handling uploads in web apps.

The official documentation for [IO](https://ruby-doc.org/3.3.3/IO.html), states:
> An instance of class IO (commonly called a stream) represents an input/output stream in the underlying operating system. 
> Class IO is the basis for input and output in Ruby.

IO being the superclass of all input/output classes in Ruby, it provides an interface that is common to many IO and IO-like classes found in Ruby or libraries.


## Their benefits, highlighted through iterative examples

#### Scenario:
> Given a log file, eligible lines must be collected, gzipped and uploaded to S3

Let's start with a basic implementation, gradually improve it using streaming and compare memory usage!


> **Note:** The complete code can be found at [https://github.com/Francois-gaspard/streaming ](https://github.com/Francois-gaspard/streaming )


-----

### Step 0: Baseline

In this baseline example, we will:
- Read the input file in memory
- Collect eligible entries in memory
- Compress eligible entries in memory
- Upload the compressed file

```ruby
@input_location = 'log/18.log'                   # Local file
@output_location = 'log/eligible_entries.log.gz' # S3 key

def compress(data)
  ActiveSupport::Gzip.compress(data)
end

def upload(data)
  s3 = Aws::S3::Client.new
  s3.put_object(bucket: ENV['AWS_BUCKET'], 
                key: @output_location, 
                body: data)
end


eligible_entries = []

file = File.read(@input_location)              # Read the file in memory
file.split(/\n/).each do |line|                # Split the file into lines
  eligible_entries << line if eligible?(line)  # Collect eligible lines
end

compressed_eligible_entries = compress(eligible_entries.join("\n"))

upload(compressed_eligible_entries)
```

#### Memory usage: 70.2 MB

-----

### Step 1: Streaming the input

File being a subclass of IO, we can use [IO#foreach](https://ruby-doc.org/3.3.3/IO.html#method-c-foreach) to iterate over the file line by line, effectively streaming its lines:
- Stream the input file 
- Collect eligible entries in memory
- Compress eligible entries in memory
- Upload the compressed file

```ruby

# ...
# This iterates over the file line by line, effectively streaming its lines:
File.foreach(input_location) do |line|           
  eligible_entries << line if eligible?(line)
end
# ...
```
#### Memory usage: 50.06 MB

-----

### Step 2: Streaming eligible entries to the compressor

Zlib::GzipWriter offers a [streaming API](https://ruby-doc.org/3.3.3/exts/zlib/Zlib/GzipFile.html#method-c-wrap) that can be used to compress data incrementally as it is generated:
- Stream the input file
- Stream eligible entries to the compressor
- Upload the compressed file

```ruby
  StringIO.open do |io|
    Zlib::GzipWriter.wrap(io) do |compressed_eligible_entries|
      File.foreach(@input_location) do |line|
        compressed_eligible_entries << line if eligible?(line)
      end
    end
    upload(io.string)
  end
```

#### Memory usage: 32.52 MB

-----

### Step 3: Streaming the compressed data to S3

AWS::S3::Object also offers a [streaming API](https://docs.aws.amazon.com/sdk-for-ruby/v3/api/Aws/S3/Object.html#upload_stream-instance_method) that can be used to upload data incrementally, as it is compressed:

- Stream the input file
- Stream eligible entries to the compressor
- Stream compressed data to S3

```ruby

def s3
  Aws::S3::Object.new(ENV.fetch('AWS_BUCKET'),
                      @output_location,
                      client: Aws::S3::Client.new)
end

s3.upload_stream(tempfile: true) do |stream|
  Zlib::GzipWriter.wrap(stream) do |compressed_eligible_entries|
    File.foreach(@input_location) do |line|
      compressed_eligible_entries << line if eligible?(line)
    end
  end
end
```

#### Memory usage: 17.25 MB

In this last example, the whole file is never loaded in memory all at once, making it possible to process files of any size with minimal impact on memory usage.

### Comparison

- Step 0: Baseline, all data loaded in memory
- Step 1: Streaming the input file, line by line
- Step 2: Streaming eligible entries to the compressor
- Step 3: Streaming compressed data to S3

```mermaid
gantt
    title Memory usage comparison
    dateFormat  X
    axisFormat %s

    section Step 0
        70.2 MB   : 0, 70
    section Step 1
        50.06 MB   : 0, 50
    section Step 2
        32.52 MB   : 0, 32
    section Step 3
        17.25 MB    : 0, 17
```
