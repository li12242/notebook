---
layout: post
title: 使用 VTune 对 MPI 应用负载平衡分析和优化
date: 2024-12-9
categories: tools
---

# 负载平衡监控

VTune 监控结果中，通过`-group-by=`参数可将MPI应用监控结果进行分类。可用参数包括：

```bash
$ vtune -report hotspots -r tidal2-n1p64-vtune-hotspots.spr-cu02/ -group-by=process,function -csv-delimiter=, > vtune_perf_fvcom_tidal2_np64.csv
```

需要仅筛选单个函数时，可以加入 `-filter=` 参数，如下所示：

```bash
vtune -report hotspots -r tidal2-n1p64-vtune-hotspots.spr-cu02/ -group-by=process -filter=function=advave_edge_gcn -csv-delimiter=, > vtune_perf_advave_edge_gcn_fvcom_tidal2_np64.csv
```