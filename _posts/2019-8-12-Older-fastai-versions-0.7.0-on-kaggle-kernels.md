---
layout: post
title: Using older fastai (< v1.0) versions on Kaggle kernels
---

If you're trying to work through fast.ai's [Introduction to Machine Learning for Coders](http://course18.fast.ai/ml) in 2019 on a [Kaggle Kernel](https://www.kaggle.com/kernels), you'll find that the fastai library api has diverged significantly.

As of Aug 2019, the fastai version on Kaggle kernels is >=1.0, but the course as of this writing uses fastai `0.7.0` Unfortunately, there's no upper range specified in the `requirements.txt` for the `0.7.0` release so if you just try and uninstall and reinstall via `!pip install fastai==0.7.0`, you'll end up pulling incompatible versions of `torchvision` and `torchtext`.

Instead, you'll have to do something like:
```
!pip install 'fastai==0.7.0' 'torchtext==0.2.3' 'torchvision==0.2.0'
```

You'll then be able to access the older api methods like `proc_df` to following along lessons. Given the pace of development on the library though, it's probably a good idea to familiarise youself with the newer apis by going through the newer courses like [Practical Deep Learning for Coders](https://course.fast.ai)

Happy coding!