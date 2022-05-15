---
layout: post
title: "Lorem Ipsum post"
author: "Inela"
published: false
---

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

{% highlight rust  linenos %}
/// Light cache structure
impl Light {
    pub fn new_with_builder(builder: &CacheBuilder, block_height: u64) -> Self {
        let cache = builder.new_cache(block_height);

        Light {
            block_height,
            cache,
        }
    }

    /// Calculate the light boundary data
    /// `header_hash` - The header hash to pack into the mix
    /// `nonce` - The nonce to pack into the mix
    pub fn compute(&self, header_hash: &H256, nonce: u64) -> H256 {
        light_compute(self, header_hash, nonce)
    }
}

#[allow(dead_code)]
pub fn slow_hash_block_height(block_height: u64) -> H256 {
    SeedHashCompute::resume_compute_seedhash([0u8; 32], 0, stage(block_height))
}
{% endhighlight %}