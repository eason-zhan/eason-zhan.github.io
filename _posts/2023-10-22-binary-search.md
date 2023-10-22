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
  int lo = 0, hi = len - 1;

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

Why the above version is not good enough is because the presences of branches([breach penalty of instruction pipeline](https://en.wikipedia.org/wiki/Instruction_pipelining)) and data cache misses can negatively impact performance. It is important to eliminate branches and data cache misses where possibile. One aliternative solution is to utilize the modified `std::lower_bound` as a replacement for `std::binary_search`, as illustrated below one by one:

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

### Classrooom vs Realworld benchmark

<div id="binary_search_benchmark"> kid </div>

<script type="text/javascript">
Highcharts.chart('binary_search_benchmark', {

    title: {
        text: 'U.S Solar Employment Growth',
        align: 'left'
    },

    subtitle: {
        text: 'By Job Category. Source: <a href="https://irecusa.org/programs/solar-jobs-census/" target="_blank">IREC</a>.',
        align: 'left'
    },

    yAxis: {
        title: {
            text: 'Number of Employees'
        }
    },

    xAxis: {
        accessibility: {
            rangeDescription: 'Range: 2010 to 2020'
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
            pointStart: 2010
        }
    },

    series: [{
        name: 'Installation & Developers',
        data: [43934, 48656, 65165, 81827, 112143, 142383,
            171533, 165174, 155157, 161454, 154610]
    }, {
        name: 'Manufacturing',
        data: [24916, 37941, 29742, 29851, 32490, 30282,
            38121, 36885, 33726, 34243, 31050]
    }, {
        name: 'Sales & Distribution',
        data: [11744, 30000, 16005, 19771, 20185, 24377,
            32147, 30912, 29243, 29213, 25663]
    }, {
        name: 'Operations & Maintenance',
        data: [null, null, null, null, null, null, null,
            null, 11164, 11218, 10077]
    }, {
        name: 'Other',
        data: [21908, 5548, 8105, 11248, 8989, 11816, 18274,
            17300, 13053, 11906, 10073]
    }],

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



