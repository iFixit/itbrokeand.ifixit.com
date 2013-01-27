---
layout: with-comments
title: "Unlimited-size Uploads in PHP"
author: Daniel Beardsley
author_url: http://github.com/danielbeardsley
summary: Unlimited-size uploads in PHP
         and how the upload_max_filesize and post_max_size php.ini settings are NOT what they seem.
---

### The jist
Setting up Apache and PHP to allow unlimited-size uploads (with respect to memory usage) is fairly easy.
Despite the ridiculously large amount of misinformation from the docs
and the the rest of the internet, it is possible.
For accepting large uploads or using newer uploading methods,
PHP's default file upload handling will have to be supplemented.

