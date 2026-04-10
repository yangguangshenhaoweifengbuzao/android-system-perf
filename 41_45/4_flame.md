### 火焰图
#### 1. 采集数据
  adb shell "simpleperf record -p 15136 -g --duration 5 -o data/local/tmp/perf.data"

#### 2. 生成 Perfetto 可用的 protobuf 格式
  adb shell "simpleperf report-sample -i data/local/tmp/perf.data --show-callchain --protobuf -o data/local/tmp/perf_record.proto"

#### 3. 拉取到本地
  adb pull data/local/tmp/perf_record.proto "c:/Users/hao.zixun/Desktop/bug/perf_record.proto"

#### 也通过as 的profiler进行查看
  ```
package com.example.myapplication;

/**
 * 火焰图分析演示代码
 *
 * 这个示例包含多层嵌套调用和不同类型的耗时操作，
 * 用于学习如何分析火焰图定位性能问题。
 *
 * 使用方法:
 * 1. 在 Activity 中调用 FlameGraphDemo.runDemo()
 * 2. 用 simpleperf 抓取: simpleperf record -g -p <pid> --duration 10
 * 3. 生成火焰图分析热点函数
 */

public class FlameGraphDemo {

public class FlameGraphDemo {

    /**
     * 入口方法 - 模拟应用启动或页面加载场景
     */
    public static void runDemo() {
        // 模拟主流程的三个阶段
        loadData();          // 阶段1: 数据加载 (占总时间约 60%)
        processData();       // 阶段2: 数据处理 (占总时间约 30%)
        renderUI();          // 阶段3: UI渲染 (占总时间约10%)
    }

    // ==================== 阶段1: 数据加载 ====================

    /**
     * loadData - 火焰图中会很宽，因为最耗时
     *
     * [问题] 模拟了大量不必要的数据加载
     */
    private static void loadData() {
        loadFromCache();     // 缓存加载
        loadFromNetwork();   // 网络加载 - 最耗时的子调用
        loadFromDatabase();  // 数据库加载
    }

    private static void loadFromCache() {
        simulateWork(100);    // 模拟缓存读取
    }

    /**
     * loadFromNetwork - 火焰图热点之一
     *
     * [问题] 模拟了同步网络请求阻塞主线程
     */
    private static void loadFromNetwork() {
        parseJsonData();     // JSON解析 
- 占网络加载 80% 时间
        validateData();      // 数据校验
    }

    /**
     * parseJsonData - 火焰图中最宽的块之一
     *
     * [问题] 模拟了低效的 JSON 解析方式
     */
    private static void parseJsonData() {
        for (int i = 0; i < 600; i++) {
            parseSingleItem(i);  // 循环解析每个 item
        }
  }

    /**
     * parseSingleItem - 火焰图热点函数
     *
     * [问题] 每次循环都创建新对象，造成内存抖动
     */
    private static void parseSingleItem(int index) {
        String data = generateFakeJson(index);  // 生成假 JSON
        extractValue(data);                     // 提取值
    }

     private static String generateFakeJson(int index) {
        // 模拟 JSON 字符串拼接
        StringBuilder sb = new StringBuilder();
        sb.append("{\"id\":").append(index);
        sb.append(",\"name\":\"item").append(index).append("\"");
        sb.append(",\"value\":").append(index * 100);
    sb.append("}");
        return sb.toString();
    }

    private static void extractValue(String json) {
        // 模拟低效的字符串解析 (不用真正的 JSON 解析器)
        int start = json.indexOf(":") + 1;
        int end = json.indexOf(",");
        simulateWork(2);  // 模拟解析耗时
    }

    private static void validateData() {
        simulateWork(200);
    }

    private static void loadFromDatabase() {
        queryDatabase();    // 数据库查询
        mapToModel();       // ORM映射
    }

    private static void queryDatabase() {
        simulateWork(300);
    }

    private static void mapToModel() {
        simulateWork(200);
    }

    // ==================== 阶段2: 数据处理 ====================

    /**
     * processData - 火焰图中较宽
     *
     * [问题] 模拟了低效的数据处理算法
     */
    private static void processData() {
        sortData();         // 排序 - 占处理时间 70%
       filterData();       // 过滤
        transformData();    // 转换
    }

    /**
     * sortData - 火焰图热点
     *
     * [问题] 模拟了 O(n^2) 的低效排序
     */
    private static void sortData() {
        bubbleSort();  // 使用冒泡排序而非快速排序
    }

    /**
     * bubbleSort - 火焰图热点函数
     *
     * [问题] O(n^2) 算法，在大数据量下非常慢
     */
    private static void bubbleSort() {
        int n = 70;
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n - i - 1; j++) {
                compareAndSwap(i, j);
            }
        }
    }

    private static void compareAndSwap(int i, int j) {
        simulateWork(1);  // 每次比较模拟微小耗时
    }

    private static void filterData() {
        for (int i = 0; i < 50; i++) {
            applyFilterRule(i);
        }
    }

    private static void applyFilterRule(int index) {
        simulateWork(3);
    }

    private static void transformData() {
        simulateWork(150);
    }

    // ==================== 阶段3: UI渲染 ====================

    /**
     * renderUI - 火焰图中较窄
     */
    private static void renderUI() {
        measureLayout();    // 测量布局
        drawView();         // 绘制
    }

    private static void measureLayout() {
        simulateWork(100);
    }

    private static void drawView() {
        for (int i = 0; i < 20; i++) {
            drawSingleView(i);
        }
    }

    private static void drawSingleView(int index) {
        simulateWork(10);
    }

    // ========== 工具方法 ====================

    /**
     * 模拟 CPU 工作
     * @param ms 模拟的毫秒数
     */
    private static void simulateWork(int ms) {
        // 用空循环模拟 CPU 消耗
        long start = System.nanoTime();
        long end = start + ms * 1_000_000L;
        while (System.nanoTime() < end) {
      // 空循环消耗 CPU
            Math.sqrt(Math.random());  // 一些无意义计算
        }
    }

    /**
     * 第二个演示方法 - 内存抖动场景
     *
     * [问题] 在循环中频繁创建对象，触发 GC
     */
    public static void runMemoryChurnDemo() {
        for (int i = 0; i < 10000; i++) {
            createTemporaryObject();
     }
    }

    private static void createTemporaryObject() {
        // 每次调用都创建新对象，很快被 GC
        byte[] data = new byte[1024];
        processDataChunk(data);
    }

    private static void processDataChunk(byte[] data) {
        simulateWork(1);
    }

    /**
     * 第三个演示方法 - 锁竞争场景
*/
    private static final Object lock = new Object();

    public static void runLockContentionDemo() {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                lockedOperationA();
            }
        });

        Thread t2 = new Thread(() -> {
        for (int i = 0; i < 100; i++) {
                lockedOperationB();
            }
        });

        t1.start();
        t2.start();

        try {
            t1.join();
            t2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private static void lockedOperationA() {
        synchronized (lock) {
            simulateWork(50);
        }
    }

    private static void lockedOperationB() {
        synchronized (lock) {
            simulateWork(50);
        }
    }
}
/*
==================== 火焰图分析指南 ====================

生成火焰图后，你应该能看到以下特点：

1. 【最宽的栈块】parseJsonData / parseSingleItem
   - 问题: 循环 1000 次，每次创建新对象
   - 优化: 使用真正的 JSON 解析器，减少对象创建

2. 【第二宽的栈块】bubbleSort / compareAndSwap
   - 问题: O(n^2) 冒泡排序，500*500=250000 次比较
   - 优化: 使用 Collections.sort() 或快速排序

3. 【调用栈层级】
   runDemo → loadData → loadFromNetwork → parseJsonData → parseSingleItem

   火焰图纵轴显示调用深度，可以看清函数调用关系

4. 【热点函数识别】
   - 红色/橙色区块 = CPU 消耗多
   - 点击区块可以放大查看详细调用栈

5. 【对比分析】
   - runDemo() 顶部宽度 = 总耗时
   - 子函数宽度占比 = 该函数耗时占比
   - parseJsonData 最宽 = 主要瓶颈

==================== 优化建议 ====================

根据火焰图，优先级：

P0: parseJsonData (占 60% 时间)
    → 使用 Gson/Moshi 替代手动解析
    → 预解析，不要在主线程解析

P1: bubbleSort (占 20% 时间)
    → 使用高效排序算法
    → 后台线程处理

P2: loadFromNetwork 同步调用
    → 改为异步网络请求

*/
  ```
