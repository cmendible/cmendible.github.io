---
author: Carlos Mendible
categories:
- dotNet
date: "2010-05-06T14:08:00Z"
description: NHibernate.Linq Issue when a query is executed having the same alias
  but with different associations paths
# tags: ["Linq", "NHibernate"]
title: NHibernate.Linq Issue when a query is executed having the same alias but with
  different associations paths
---
When running a query like the following:

``` chsharp 
var query = nhib.Arms.Where(a => a.LeftHand.Thumb.Length == 1 || a.RightHand.Thumb.Length == 1);
```

Thumb alias was always taken as part of the a.LeftHand association path, therefore leading to wrong results.

I've worked on patch and test to fix this issue, so Thumb alias is once part of the a.LeftHand association path and once as part of the a.RightHand association path.

You can find a patch file for this issue here: <http://github.com/cmendible/nhibernate-contrib/downloads#download_31118>

Or you can also download pruiz's already patched version of nhiberante-contribut project here: <http://github.com/pruiz/nhibernate-contrib/commit/eada73cce086a6457e5e64b0413b97a8f53863ac>

Hope it helps!!!