### 简介

用 `Go` 编写的 `benchmark` 输出解析器，功能如下

* 读取 `go test -bench=. -benchmem` 的输出文件（如 `benchmark.txt`）

* 解析出每行数据

* 写入成 `CSV` 文件（如 `benchmark.csv`）

* `Web UI` 可视化数据

### 仅Go解析器

```go
package main

import (
	"bufio"
	"encoding/csv"
	"fmt"
	"os"
	"regexp"
)

type BenchmarkResult struct {
	Name        string
	Runs        string
	NsPerOp     string
	BytesPerOp  string
	AllocsPerOp string
}

func main() {
	inputFile := "benchmark.txt"
	outputFile := "benchmark.csv"

	file, err := os.Open(inputFile)
	if err != nil {
		fmt.Println("读取文件失败:", err)
		return
	}
	defer file.Close()

	// 匹配 benchmark 输出行
	re := regexp.MustCompile(`^Benchmark(\w+)-\d+\s+(\d+)\s+(\d+)\s+ns/op\s+(\d+)\s+B/op\s+(\d+)\s+allocs/op`)

	var results []BenchmarkResult

	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		line := scanner.Text()
		matches := re.FindStringSubmatch(line)
		if len(matches) == 6 {
			results = append(results, BenchmarkResult{
				Name:        matches[1],
				Runs:        matches[2],
				NsPerOp:     matches[3],
				BytesPerOp:  matches[4],
				AllocsPerOp: matches[5],
			})
		}
	}

	if err := scanner.Err(); err != nil {
		fmt.Println("读取行失败:", err)
		return
	}

	// 写入 CSV
	outFile, err := os.Create(outputFile)
	if err != nil {
		fmt.Println("创建 CSV 文件失败:", err)
		return
	}
	defer outFile.Close()

	writer := csv.NewWriter(outFile)
	defer writer.Flush()

	headers := []string{"name", "runs", "ns_per_op", "bytes_per_op", "allocs_per_op"}
	if err := writer.Write(headers); err != nil {
		fmt.Println("写入 CSV 头失败:", err)
		return
	}

	for _, r := range results {
		row := []string{r.Name, r.Runs, r.NsPerOp, r.BytesPerOp, r.AllocsPerOp}
		if err := writer.Write(row); err != nil {
			fmt.Println("写入行失败:", err)
			return
		}
	}

	fmt.Println("✅ 成功导出 CSV:", outputFile)
}
```

#### 使用方法

* 保存为 `main.go`

* 编译运行：

```shell
go run main.go
```

或编译为可执行文件：

```shell
go build -o benchparser
./benchparser
```

确保 `benchmark.txt` 在同目录，输出会生成 `benchmark.csv`。

### 使用 Web UI 可视化数据

#### 功能需求

* 支持输入：
    * 从 `benchmark.txt` 文件读取
    * 从标准输入读取（如管道输入）
* 解析 `go test -bench=. -benchmem` 输出
* 提供一个 `Web UI`：
    * 显示 `ns/op`、`B/op`、`allocs/op` 的柱状图
    * 支持切换不同维度视图

#### 项目结构设计（Go + HTML + JS）

```
benchmark-visualizer/
├── main.go                 ← 启动解析器 + Web Server
├── static/
│   ├── index.html          ← Web 页面
│   ├── chart.js            ← Chart.js 库
│   └── main.js             ← 渲染图表的 JS 脚本
```

#### 第一步：Go 服务器（main.go）

```go
package main

import (
	"bufio"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"regexp"
)

type BenchmarkResult struct {
	Name        string `json:"name"`
	Runs        int    `json:"runs"`
	NsPerOp     float64`json:"ns_per_op"`
	BytesPerOp  int    `json:"bytes_per_op"`
	AllocsPerOp int    `json:"allocs_per_op"`
}

var results []BenchmarkResult

func parseBenchmark(input *os.File) ([]BenchmarkResult, error) {
	re := regexp.MustCompile(`^(Benchmark\S*)\s+(\d+)\s+([\d\.]+)\s+ns/op\s+(\d+)\s+B/op\s+(\d+)\s+allocs/op`)
	scanner := bufio.NewScanner(input)
	var parsed []BenchmarkResult

	for scanner.Scan() {
		line := scanner.Text()
        matches := re.FindStringSubmatch(line)
        if len(matches) == 6 {
            var r BenchmarkResult
            r.Name = matches[1]
            fmt.Sscanf(matches[2], "%d", &r.Runs)
            fmt.Sscanf(matches[3], "%f", &r.NsPerOp)
            fmt.Sscanf(matches[4], "%d", &r.BytesPerOp)
            fmt.Sscanf(matches[5], "%d", &r.AllocsPerOp)
            parsed = append(parsed, r)
        }
	}

	return parsed, scanner.Err()
}

func dataHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(results)
}

func main() {
	// 判断是否有标准输入
	stat, _ := os.Stdin.Stat()
	var err error

	if (stat.Mode() & os.ModeCharDevice) == 0 {
		// 从标准输入读取
		results, err = parseBenchmark(os.Stdin)
	} else {
		// 默认读取 benchmark.txt 文件
		file, err := os.Open("benchmark.txt")
		if err != nil {
			fmt.Println("打开 benchmark.txt 失败:", err)
			return
		}
		defer file.Close()
		results, err = parseBenchmark(file)
	}

	if err != nil {
		fmt.Println("解析失败:", err)
		return
	}

	fs := http.FileServer(http.Dir("./static"))
	http.Handle("/", fs)
	http.HandleFunc("/data", dataHandler)

	fmt.Println("🚀 服务器已启动：http://localhost:8080")
	http.ListenAndServe(":8080", nil)
}
```

#### 第二步：Web 页面（static/index.html）

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Go Benchmark 可视化</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <script src="main.js" defer></script>
</head>
<body>
  <h2>Go Benchmark 可视化结果</h2>
  <select id="metric-select">
    <option value="ns_per_op">ns/op</option>
    <option value="bytes_per_op">B/op</option>
    <option value="allocs_per_op">allocs/op</option>
  </select>
  <canvas id="benchmarkChart" width="900" height="500"></canvas>
</body>
</html>
```

#### 第三步：Chart 渲染脚本（static/main.js）

```javascript
let chart;
const ctx = document.getElementById('benchmarkChart').getContext('2d');

function renderChart(data, metric = "ns_per_op") {
  const labels = data.map(item => item.name);
  const values = data.map(item => item[metric]);

  if (chart) chart.destroy();

  chart = new Chart(ctx, {
    type: 'bar',
    data: {
      labels: labels,
      datasets: [{
        label: metric,
        data: values,
        backgroundColor: 'rgba(54, 162, 235, 0.6)'
      }]
    },
    options: {
      scales: {
        y: {
          beginAtZero: true
        }
      }
    }
  });
}

fetch('/data')
  .then(res => res.json())
  .then(data => {
    renderChart(data);
    document.getElementById('metric-select').addEventListener('change', (e) => {
      renderChart(data, e.target.value);
    });
  });
```

#### 运行方式

* 方法 1：从文件读取

```shell
go run main.go
# 或
go build -o benchvis && ./benchvis
```

* 方法 2：从标准输入读取

```shell
go test -bench=. -benchmem | go run main.go
```

然后访问：`http://localhost:8080`

效果如图：

![alt text](/images/go/go-benchmark-image.png)

#### benchmark.txt文件示例

```
goos: darwin
goarch: arm64
pkg: echo
cpu: Apple M1
BenchmarkEchoAll1-8   	  271656	      4631 ns/op	   21080 B/op	      99 allocs/op
BenchmarkEchoAll2-8   	  224186	      5874 ns/op	   31560 B/op	      99 allocs/op
BenchmarkEchoAll3-8   	 1273263	       836.1 ns/op	     640 B/op	       1 allocs/op
BenchmarkEchoAll4-8   	 1457311	       809.8 ns/op	    1912 B/op	       8 allocs/op
PASS
ok  	echo	9.128s
```