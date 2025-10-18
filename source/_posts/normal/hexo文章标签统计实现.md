---
title: hexo文章标签统计实现
date: 2020-04-24 22:58:00
categories: 随笔
tags:
cover: https://static.jiangliuhong.top/blogimg/other/hexo-logo.jpg
---

# hexo文章标签统计实现

> 近日浏览公司大神[闪烁之狐](https://blinkfox.github.io/)的博客，发现他的关于页有着一个很漂亮的文章统计图，于是想给自己的博客也安排上。

## 效果预览

先附上我的效果图：[https://jiangliuhong.top/statistics/](https://jiangliuhong.top/statistics/)

![统计图](https://static.jiangliuhong.top/blogimg/other/hexo_state.png)

再附上大佬的效果图：[https://blinkfox.github.io/about/](https://blinkfox.github.io/about/)

![统计图2](https://static.jiangliuhong.top/blogimg/other/blink_state.png)

## 实现思路

### 统计数据来源

- 文章发布统计：利用`site.posts`获取文章信息

```javascript
site.posts.forEach(function (post) {
    var month = post.date.format('YYYY-MM');
    if (monthMap.has(month)) {
        monthMap.set(month, monthMap.get(month) + 1);
    }
});
```

- 文章分类统计：利用`site.categories`获取分类信息

```javascript
    site.categories.map(function(category) {
        categoryArr.push({'name': category.name, 'value': category.length})
    });
```

- 标签统计：利用`site.tags`获取标签信息

```javascript
    site.tags.map(function(tag) {
        tagArr.push({'name': tag.name, 'value': tag.length});
    });
```

### 统计呈现方式

使用[echarts](https://echarts.apache.org/examples/zh/index.html)呈现，具体参见[echarts](https://echarts.apache.org/examples/zh/index.html)文档

### 新建layout

在`layout`中新建`statistics`布局，然后在`source`目录新建`statistics/index.md`文件，在文件中指定`layout`。

```
---
title: 我的统计
date: 20120-04-22 12:35:59
layout: statistics
---

```

## 具体实现

> 下面的实现访问为基于[hexo-fluid](https://github.com/fluid-dev/hexo-theme-fluid)主题实现

首先在`layout`目录下新建一个`statistics.ejs`文件，文件内容为：

```
<%
page.layout = "statistics"
page.banner_img = theme.statistics.banner_img
page.banner_img_height = theme.statistics.banner_img_height
page.banner_mask_alpha = theme.statistics.banner_mask_alpha
%>
<div class="mt-5 markdown-body">
    <%- partial('_widget/mystatistics') %>
</div>
```

在`_widget`目录下创建一个`mystatistics.ejs`，文件内容为：

> 需要注意的是，在下面源码中有这样一行`src="<%- url_for(theme.my.libs.js.echarts) %>"`，这是指定了一个echarts.js文件地址，你可以前往官网[https://echarts.apache.org/zh/download.html](https://echarts.apache.org/zh/download.html)下载（同时也可以访问我的地址进行下载[https://jiangliuhong.top/js/echarts/echarts.min.js](https://jiangliuhong.top/js/echarts/echarts.min.js)），然后把这个修改成你的地址即可。

```
<style type="text/css">
    #posts-chart,
    #categories-chart,
    #tags-chart {
        width: 100%;
        height: 300px;
        margin: 0.5rem auto;
        padding: 0.5rem;
    }
</style>

<div id="postCharts" class="post-charts">
   <div class="title center-align center-align-title" data-aos="zoom-in-up">
        文章统计图
    </div>
    <div class="row">
        <div class="chart col s12 m6 l4" data-aos="zoom-in-up">
            <div id="posts-chart"></div>
        </div>
        <div class="chart col s12 m6 l4" data-aos="zoom-in-up">
            <div id="categories-chart"></div>
        </div>
    </div>
    <div class="row">
        <div class="chart col s12 m6 l4" data-aos="zoom-in-up">
            <div id="tags-chart"></div>
        </div>
    </div>
</div>

<script type="text/javascript" src="<%- url_for(theme.my.libs.js.echarts) %>"></script>
<script>
    let postsChart = echarts.init(document.getElementById('posts-chart'));
    let categoriesChart = echarts.init(document.getElementById('categories-chart'));
    let tagsChart = echarts.init(document.getElementById('tags-chart'));

    <%
    /* calculate postsOption data. */
    var startDate = moment().subtract(1, 'years').startOf('month');
    var endDate = moment().endOf('month');

    var monthMap = new Map();
    var dayTime = 3600 * 24 * 1000;
    for (var time = startDate; time <= endDate; time += dayTime) {
        var month = moment(time).format('YYYY-MM');
        if (!monthMap.has(month)) {
            monthMap.set(month, 0);
        }
    }

    // post and count map.
    site.posts.forEach(function (post) {
        var month = post.date.format('YYYY-MM');
        if (monthMap.has(month)) {
            monthMap.set(month, monthMap.get(month) + 1);
        }
    });

    // xAxis data and yAxis data.
    var monthArr = JSON.stringify([...monthMap.keys()]);
    var monthValueArr = JSON.stringify([...monthMap.values()]);
    %>

    let postsOption = {
        title: {
            text: '文章发布统计图',
            top: -5,
            x: 'center'
        },
        tooltip: {
            trigger: 'axis'
        },
        xAxis: {
            type: 'category',
            data: <%- monthArr %>
        },
        yAxis: {
            type: 'value',
        },
        series: [
            {
                name: '文章篇数',
                type: 'line',
                color: ['#6772e5'],
                data: <%- monthValueArr %>,
                markPoint: {
                    symbolSize: 45,
                    color: ['#fa755a','#3ecf8e','#82d3f4'],
                    data: [{
                        type: 'max',
                        itemStyle: {color: ['#3ecf8e']},
                        name: '最大值'
                    }, {
                        type: 'min',
                        itemStyle: {color: ['#fa755a']},
                        name: '最小值'
                    }]
                },
                markLine: {
                    itemStyle: {color: ['#ab47bc']},
                    data: [
                        {type: 'average', name: '<%- __("average")  %>'}
                    ]
                }
            }
        ]
    };

    <%
    /* calculate categoriesOption data. */
    var categoryArr = [];
    site.categories.map(function(category) {
        categoryArr.push({'name': category.name, 'value': category.length})
    });

    var categoryArrJson = JSON.stringify(categoryArr);
    %>

    let categoriesOption = {
        title: {
            text: '文章分类统计图',
            top: -4,
            x: 'center'
        },
        tooltip: {
            trigger: 'item',
            formatter: "{a} <br/>{b} : {c} ({d}%)"
        },
        series: [
            {
                name: '分类',
                type: 'pie',
                radius: '50%',
                color: ['#6772e5', '#ff9e0f', '#fa755a', '#3ecf8e', '#82d3f4', '#ab47bc', '#525f7f', '#f51c47', '#26A69A'],
                data: <%- categoryArrJson %>,
                itemStyle: {
                    emphasis: {
                        shadowBlur: 10,
                        shadowOffsetX: 0,
                        shadowColor: 'rgba(0, 0, 0, 0.5)'
                    }
                }
            }
        ]
    };

    
    <%
    /* calculate tagsOption data. */
    // get all tags name and count,then order by length desc.
    var tagArr = [];
    site.tags.map(function(tag) {
        tagArr.push({'name': tag.name, 'value': tag.length});
    });
    tagArr.sort((a, b) => {return b.value - a.value});

    // get Top 10 tags name and count.
    var tagNameArr = [];
    var tagCountArr = [];
    for (var i = 0, len = Math.min(tagArr.length, 10); i < len; i++) {
        tagNameArr.push(tagArr[i].name);
        tagCountArr.push(tagArr[i].value);
    }

    var tagNameArrJson = JSON.stringify(tagNameArr);
    var tagCountArrJson = JSON.stringify(tagCountArr);

    %>

    let tagsOption = {
        title: {
            text: 'TOP10 标签统计图',
            top: -5,
            x: 'center'
        },
        tooltip: {},
        xAxis: [
            {
                type: 'category',
                axisLabel:{
                    interval:0,
                    rotate:40
                },
                data: <%- tagNameArrJson %>
            }
        ],
        yAxis: [
            {
                type: 'value'
            }
        ],
        series: [
            {
                name: 'TOP10标签统计',
                type: 'bar',
                color: ['#82d3f4'],
                barWidth : 18,
                data: <%- tagCountArrJson %>,
                markPoint: {
                    symbolSize: 45,
                    data: [{
                        type: 'max',
                        itemStyle: {color: ['#3ecf8e']},
                        name: '最大值'
                    }, {
                        type: 'min',
                        itemStyle: {color: ['#fa755a']},
                        name: '最小值'
                    }],
                },
                markLine: {
                    itemStyle: {color: ['#ab47bc']},
                    data: [
                        {type: 'average', name: '平均值'}
                    ]
                }
            }
        ]
    };

    // render the charts
    postsChart.setOption(postsOption);
    categoriesChart.setOption(categoriesOption);
    tagsChart.setOption(tagsOption);
</script>

```