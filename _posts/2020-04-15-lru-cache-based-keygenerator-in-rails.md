---
layout: post
title: "LRU-Cache based KeyGenerator in Rails"
date: 2020-04-15 00:00:00 +0530
tags: coding
---

The Active Support component of Ruby on Rails provides a class named KeyGenerator for generating secret keys. This is a wrapper around OpenSSL’s implementation of PBKDF2 (Password-Based Key Definition Function 2) and is commonly used to generate secrets keys for encryption use-cases.

`CachingKeyGenerator` is a wrapper around `KeyGenerator` class which helps us to avoid re-executing the key generation process when it is called using the same `salt` and the `key_size`. This is helpful because the key generation process is computationally costly. However, `CachingKeyGenerator` uses an instance of `Concurrent::Map` to store the generated keys in memory to avoid the re-execution. This is not helpful when the number of keys to be cached is large so that periodic cleanup is required on least recently used ones.

Assume that we are using `CachingKeyGenerator` to generate unique secret keys for encrypting personal information of our application’s users. This will return the key if it exists in the `Concurrent::Map` or re-execute the key generation process and cache the key before returning to the caller. When the userbase is huge, it is not practical to store all the generated keys in the memory of application instances. Instead, it would be ideal to perform a periodic clean up of the cached keys to avoid memory hogging.

We will see how we can solve this issue using a new wrapper around KeyGenerator. Instead of `Concurrent::Map` in `CachingKeyGenerator`, this implementation uses `ActiveSupport::Cache::MemoryStore` that is Thread-safe and has an LRU based clean-up mechanism built-in. An excerpt from the documentation regarding the Thread Safety says

> This cache has a bounded size specified by the :size options to the initializer (default is 32Mb). When the cache exceeds the allotted size, a cleanup will occur which tries to prune the cache down to three-quarters of the maximum size by removing the least recently used entries.

<br />

{% highlight ruby %}

class LruCachingKeyGenerator
  KEY_EXPIRY = 1.day # expire the keys which are older than a day

  def initialize(key_generator)
    @key_generator = key_generator
    @keys_cache = ActiveSupport::Cache::MemoryStore.new(expires_in: KEY_EXPIRY)
  end

  # Returns a derived key suitable for use.

  def generate_key(*args)
    cached_key = @keys_cache.fetch(args.join)
    return cached_key unless cached_key.nil?

    # cache miss, generate a new key
    generated_key = @key_generator.generate_key(*args)
    @keys_cache.write(args.join, generated_key)
    return generated_key
  end
end

{% endhighlight %}

As the default cache size is 32Mb, this helps to avoid memory hogging of our application while providing a cache for frequently used keys.