---
layout: post
title: Binary Search
date: 2023-10-22 23:37 +0800
categories: [Programming, C++]
tags: [binary search, benchmark]
image:
  path: /assets/img/decoys/binary_search_tree.jpeg
  alt: A Full Binary Tree
---

<script src="https://code.highcharts.com/highcharts.js"></script>

Binary search is an efficient algorithm used to determine if a target value exists within a sorted array. It is staightforward to implement and can be optimized for improved performance. In the upcoming section of this blog, I will present a comprehensive explanation of both the basic version and the optimized versions.

### Classroom approach

As we learned from classroom lectures, writing a binary search becomes a straightforward task, as demonstrated below:

```c++
int binary_search(const int *data, int len, const int &x) {
  long lo = 0, hi = len - 1;

  while (lo <= hi) {
    int mid = (lo + hi) >> 1;
    if (data[mid] == x) {
      return mid;
    } else if (data[mid] < x) {
      lo = mid + 1;
    } else {
      hi = mid - 1;
    }
  }

  return -1;
}
```

It exhibits an efficient time complexity of O(log n). However, in high-performance, low-latency systems, it may be necessary to rewrite it into a more optimized version.

### Real-world scenarios

Why the above version is not good enough is because the presences of branches([breach penalty of instruction pipeline](https://en.wikipedia.org/wiki/Instruction_pipelining)) and data cache misses can negatively impact performance. It is important to eliminate branches and data cache misses where possibile.

#### Using `std::lower_bound()` versions

One aliternative solution is to utilize the modified `std::lower_bound` as a replacement for `std::binary_search`, as illustrated below one by one:

```c++
int binary_search(const int *data, int len, const int &x) {
  int i = lower_bound(data, len, x);

  return (i == len || data[i] != x) ? -1 : i;
}
```

- `lower_bound` v1

```c++
int lower_bound(const int *data, int len, const int &x) {
  if (!len || x > data[len - 1])
    return len;

  const int *base = data;
  while (len > 1) {
    int half = (len >> 1);
    if (base[half - 1] < x) {
      base += half;
      len -= half;
    } else {
      len -= half;
    }
  }

  return base - data;
}
```

- `lower_bound` v2 (branchless)

```c++
int lower_bound_v2(const int *data, int len, const int &x) {
  if (!len || x > data[len - 1])
    return len;

  const int *base = data;
  while (len > 1) {
    int half = (len >> 1);
    base += half * (base[half - 1] < x);
    len -= half;
  }

  return base - data;
}
```

- `lower_bound` v3 (branchless with prefetch)

```c++
int lower_bound_v3(const int *data, int len, const int &x) {
  if (!len || x > data[len - 1])
    return len;

  const int *base = data;
  while (len > 1) {
    int half = (len >> 1);
    __builtin_prefetch(base + (half >> 1) - 1);
    __builtin_prefetch(base + half + (half >> 1) - 1);
    base += half * (base[half - 1] < x);
    len -= half;
  }

  return base - data;
}
```

#### Using Eytzinger Layout version

In the paper ["Arrays layouts for comparison-based searching"](https://arxiv.org/pdf/1509.05053.pdf) by Paul-Virak Khuong and Pat Morin, they describe a specific method for optimizing binary search by reorganizing elements of a sorted array in a cache-friendly manner. To gain a detailed understanding of this technique, I recommend reading a [blog](https://en.algorithmica.org/hpc/data-structures/binary-search/) that offers an in-depth explanation on the subject.

However, it is worth noting that the blog presents a recursive version to rearrange the elements, which can lead to a crash due to the recursion stack depth limit. To address this issue, I have implemented an iterative version, which avoids the curse of recursion.

```c++
class Eytzinger {
public:
  struct zpair {
    int val = -1;
    int idx = -1;
  };

  static constexpr int block_size = 64 / sizeof(zpair);

  Eytzinger(const std::vector<int> &vec) : z_(vec.size() + 1) {
    build(vec);
  }

  int search(int target) {
    auto zp = search_(target);
    return (zp.val == target) ? zp.idx : -1;
  }

private:
  void build(const std::vector<int>& vec) {
    int i = 1;
    std::queue<std::pair<int, int>> q;
    q.push({0, vec.size()});

    while (!q.empty()) {
      auto pair = q.front();
      q.pop();
      long lo = pair.first, hi = pair.second;
      if (lo < hi) {
          int mid = (lo + hi) >> 1;
          z_[i].idx = mid;
          z_[i].val = vec[mid];
          ++i;

          q.push({lo, mid});
          q.push({mid + 1, hi});
      }
    }
  }

  inline zpair search_(int target) {
    long k = 1;
    while (k < (int)z_.size()) {
      __builtin_prefetch(z_.data() + k * block_size);
      k = (k << 1) + (z_[k].val < target);
    }
    k >>= __builtin_ffs(~k);
    return z_[k];
  }

private:
  alignas(64) std::vector<zpair> z_;
};
```

### Benchmarks

I have conducted a performance benchmark of different versions. In order to analyze the results clearly, I have divided them into two parts: one for smaller datasets and another for larger datasets. Based on the results, It can be concluded that for smaller datasets, version 2 performs better. However, for larger datasets, the eyztiner version outperforms the others. This distinction helps provide a clearer understanding of the relative performance of each version in different scenarios. Here is my [source code](https://github.com/eason-zhan/dsy/blob/main/demos/01_binary_search/binary_search_bench.cpp).

<div id="binary_search_benchmark_1"> kid </div>

<div id="binary_search_benchmark_2"> kid </div>

<script type="text/javascript">

let series = [
    {
        "type": "line",
        "name": "basic",
        "data": [
            31.480225398348725,
            32.45890810213998,
            34.170843076134034,
            36.75378336460658,
            40.27341361684449,
            42.768436951829614,
            46.25044195748869,
            51.51343385221509,
            58.33586618095851,
            65.55152901133675,
            70.88178619781584,
            76.1172028318673,
            80.40616103603496,
            87.98203314169902,
            93.79682378716186,
            99.0859962740786,
            106.56064355794693,
            117.68110714968564,
            140.76902373810492,
            161.0193936386849,
            179.1652102919735,
            212.3264790805554,
            288.139337101508,
            353.38004035628757,
            410.50606484760914,
            461.3092316875571,
            515.8968620311291,
            572.44213380408,
            622.7650714248209,
            716.7129642245164
        ]
    },
    {
        "type": "line",
        "name": "v1",
        "data": [
            32.65668978522394,
            34.682993446357365,
            36.284687141272734,
            38.850679587592694,
            40.814409107178705,
            43.18533612452534,
            46.559234363000556,
            51.81565880681424,
            59.3367869154737,
            68.88671079765936,
            75.68067008314182,
            80.73561263271897,
            85.66907278887876,
            92.9137963074892,
            98.97794063062662,
            104.38770284930267,
            112.33461871708775,
            122.50889022445104,
            144.47268148436515,
            164.21337646853723,
            181.82951148788402,
            215.48646883349716,
            290.39264564141496,
            356.83086802949674,
            414.7603140902372,
            464.98194575517266,
            516.2459229708372,
            572.9780465735431,
            623.5130249145908,
            703.7088882160764
        ]
    },
    {
        "type": "line",
        "name": "v2",
        "data": [
            33.06553681907007,
            34.45737511480519,
            35.6875599150909,
            36.81319111794954,
            38.31888038593356,
            40.122608793464785,
            42.343457962158475,
            44.55523694718221,
            46.56486891223799,
            47.97187629454105,
            49.374576470987726,
            49.986473190623734,
            51.468395543588244,
            56.35228142463439,
            60.87157978793975,
            66.48258549213553,
            75.62108363312093,
            95.44733703623855,
            133.31444106783692,
            163.04253296334088,
            188.7505447290096,
            241.7056147085185,
            367.478626285697,
            477.34938314018297,
            563.3093314803766,
            641.7997016104671,
            726.13142887982,
            827.9994560902473,
            903.8176200658786,
            1035.3305519318835
        ]
    },
    {
        "type": "line",
        "name": "v3",
        "data": [
            34.70843077282957,
            36.82895235106884,
            38.447949858065314,
            39.713862318748326,
            41.32535541452094,
            42.589380030146366,
            44.32625996693835,
            46.38572658378074,
            48.58763299873174,
            49.60503966006818,
            51.210280444301105,
            52.256247177416526,
            53.39047545007845,
            55.71912304225951,
            58.71545033862115,
            62.025227814911695,
            69.26226327075814,
            79.66042281815018,
            101.50390124085723,
            117.19049602855345,
            131.9456912305767,
            165.6076677314619,
            242.95564885843174,
            302.1949652874272,
            347.2705947047307,
            387.7043937307303,
            436.48868210465224,
            489.69110776972155,
            526.4195217077701,
            621.8119928059459
        ]
    },
    {
        "type": "line",
        "name": "eytzinger",
        "data": [
            33.24432114944493,
            34.89904498713839,
            36.680396637782835,
            38.48838887147222,
            40.46605410962306,
            44.90891510421854,
            51.818482567752866,
            58.96055573214553,
            61.60281932728481,
            59.6595466532955,
            58.33096508011295,
            57.924389033788096,
            58.965361518107564,
            61.10043160721018,
            63.920849786988654,
            71.71463756872126,
            80.46431824771052,
            73.67246286378537,
            109.97926260939265,
            88.71435777095043,
            108.31024609200827,
            171.5207537556036,
            204.86714388863632,
            235.9237166086739,
            269.20994909959916,
            301.11646966169803,
            336.1402449307145,
            378.1132094421591,
            430.89061141589735,
            479.7969589506907
        ]
    }
];

let series_copy = JSON.parse(JSON.stringify(series));

let part1_series = series_copy.map(e => {
		e.data.splice(15);
		return e;
	});

let part2_series = series.map(e => {
	e.data.splice(0, e.data.length - 15);
	return e;
});

Highcharts.chart('binary_search_benchmark_1', {
    title: {
				useHTML: true,
        text: 'Binary Search Benchmark, Elements from 2<sup>1</sup> to 2 <sup>15</sup>',
        align: 'left'
    },

    yAxis: {
        title: {
            text: 'Time(ns)'
        }
    },

    xAxis: {
        title: {
          useHTML:true,
          text: 'Elements(2<sup>n</sup>)'
        },
        accessibility: {
            rangeDescription: 'Range: 1 to 10'
        }
    },

    legend: {
        layout: 'vertical',
        align: 'right',
        verticalAlign: 'middle'
    },

    plotOptions: {
        series: {
            label: {
                connectorAllowed: false
            },
            pointStart: 1
        }
    },
		series:part1_series,
    responsive: {
        rules: [{
            condition: {
                maxWidth: 500
            },
            chartOptions: {
                legend: {
                    layout: 'horizontal',
                    align: 'center',
                    verticalAlign: 'bottom'
                }
            }
        }]
    }
});

Highcharts.chart('binary_search_benchmark_2', {
    title: {
        useHTML: true,
        text: 'Binary Search Benchmark, Elements from 2<sup>16</sup> to 2 <sup>30</sup>',
        align: 'left'
    },

    yAxis: {
        title: {
            text: 'Time(ns)'
        }
    },

    xAxis: {
        title: {
          useHTML:true,
          text: 'Elements(2<sup>n</sup>)'
        },
    },

    legend: {
        layout: 'vertical',
        align: 'right',
        verticalAlign: 'middle'
    },

    plotOptions: {
        series: {
            label: {
                connectorAllowed: false
            },
            pointStart: 16
        }
    },
		series:series,
    responsive: {
        rules: [{
            condition: {
                maxWidth: 500
            },
            chartOptions: {
                legend: {
                    layout: 'horizontal',
                    align: 'center',
                    verticalAlign: 'bottom'
                }
            }
        }]
    }
});
</script>
