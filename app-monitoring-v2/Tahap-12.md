Sip, ini bagian yang paling “kerasa jadi produk”: **Dashboard UI realtime** 🔥
Dan ya, keputusan kamu benar: JavaScript dipisah di `script.js` supaya rapi dan scalable.

---

# 🎯 Target Dashboard UI (Versi Kita)

Kita akan buat:

* CPU gauge (circle)
* RAM gauge
* SSD gauge
* Network live chart (RX/TX)
* Load average panel
* Uptime panel
* Realtime WebSocket update
* Dark modern UI (Tailwind)

---

# 📁 Struktur Frontend

Tambahkan:

```text id="ui1"
web/
├── templates/
│   └── index.html
└── static/
    ├── js/
    │   └── script.js
    └── css/
        └── style.css (opsional)
```

---

# 1. 🧠 SERVE STATIC FILE (GIN)

Update `main.go`:

```go id="ui2"
r := gin.Default()

r.Static("/static", "./web/static")
r.LoadHTMLGlob("web/templates/*")

r.GET("/", func(c *gin.Context) {
	c.HTML(200, "index.html", nil)
})

r.GET("/api/summary", api.SummaryHandler())
r.GET("/ws", api.WSHandler(hub))
```

---

# 2. 🧾 DASHBOARD HTML

Buat:

```text id="ui3"
web/templates/index.html
```

---

## 💎 UI DASHBOARD

```html id="ui4"
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Aplikasi Monitor</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>

<body class="bg-slate-900 text-white">

<div class="p-6">

    <h1 class="text-2xl font-bold mb-4">📊 Aplikasi Monitor</h1>

    <!-- GRID CARDS -->
    <div class="grid grid-cols-4 gap-4">

        <div class="bg-slate-800 p-4 rounded-xl">
            <h2>CPU</h2>
            <p id="cpu">0%</p>
        </div>

        <div class="bg-slate-800 p-4 rounded-xl">
            <h2>RAM</h2>
            <p id="ram">0%</p>
        </div>

        <div class="bg-slate-800 p-4 rounded-xl">
            <h2>SSD</h2>
            <p id="disk">0%</p>
        </div>

        <div class="bg-slate-800 p-4 rounded-xl">
            <h2>Network</h2>
            <p id="net">0 KB/s</p>
        </div>

    </div>

    <!-- LOAD + UPTIME -->
    <div class="grid grid-cols-2 gap-4 mt-4">

        <div class="bg-slate-800 p-4 rounded-xl">
            <h2>Load Average</h2>
            <p id="load">0 / 0 / 0</p>
        </div>

        <div class="bg-slate-800 p-4 rounded-xl">
            <h2>Uptime</h2>
            <p id="uptime">0s</p>
        </div>

    </div>

    <!-- CHART -->
    <div class="bg-slate-800 p-4 rounded-xl mt-6">
        <canvas id="chart"></canvas>
    </div>

</div>

<script src="/static/js/script.js"></script>

</body>
</html>
```

---

# 3. ⚡ SCRIPT WEBSOCKET (REALTIME CORE)

Buat:

```text id="ui5"
web/static/js/script.js
```

---

## 🚀 REALTIME ENGINE

```javascript id="ui6"
let ws = new WebSocket("ws://" + location.host + "/ws");

// DATA HISTORY (chart)
let cpuData = [];

let chart = new Chart(document.getElementById("chart"), {
    type: "line",
    data: {
        labels: [],
        datasets: [{
            label: "CPU %",
            data: [],
            borderColor: "lime",
            tension: 0.3
        }]
    }
});

ws.onmessage = function(event) {

    let data = JSON.parse(event.data);

    console.log(data);

    // contoh mapping (nanti kita upgrade full payload)
    if (data.cpu_usage !== undefined) {

        document.getElementById("cpu").innerText = data.cpu_usage.toFixed(2) + "%";
        document.getElementById("ram").innerText = data.ram_usage + "%";
        document.getElementById("disk").innerText = data.disk_usage.toFixed(2) + "%";

        document.getElementById("net").innerText =
            (data.rx_speed / 1024).toFixed(2) + " KB/s";

        document.getElementById("load").innerText =
            data.load1 + " / " + data.load5 + " / " + data.load15;

        document.getElementById("uptime").innerText =
            data.uptime + " sec";

        // chart update
        let now = new Date().toLocaleTimeString();

        chart.data.labels.push(now);
        chart.data.datasets[0].data.push(data.cpu_usage);

        if (chart.data.labels.length > 20) {
            chart.data.labels.shift();
            chart.data.datasets[0].data.shift();
        }

        chart.update();
    }
};
```

---

# 4. 🔥 UBAH WEBSOCKET SERVER (IMPORTANT)

Sekarang kita kirim **API summary ke WS**, bukan dummy data.

---

## UPDATE main.go WS broadcaster

Ganti bagian ini:

```go
go func() {

	for {
		data := gin.H{
			"type": "realtime",
			"time": time.Now().Unix(),
		}

		b, _ := json.Marshal(data)
		hub.Broadcast <- b

		time.Sleep(10 * time.Second)
	}
}()
```

---

## 🔥 MENJADI REAL DATA:

```go
go func() {

	for {

		summary := gin.H{}

		// ambil data real
		cpuUsage, cpuCore, _ := collector.GetCPUUsage()
		ramTotal, ramUsed, ramUsage, _ := collector.GetMemory()
		diskTotal, diskUsed, diskUsage, _ := collector.GetDisk()
		net, _ := collector.GetNetworkSpeed()
		load1, load5, load15 := system.LoadAverage()
		uptime := system.Uptime()

		summary = gin.H{
			"cpu_usage": cpuUsage,
			"cpu_core":  cpuCore,

			"ram_total": ramTotal,
			"ram_used":  ramUsed,
			"ram_usage": ramUsage,

			"disk_total": diskTotal,
			"disk_used":  diskUsed,
			"disk_usage": diskUsage,

			"rx_speed": net.RXSpeed,
			"tx_speed": net.TXSpeed,

			"load1": load1,
			"load5": load5,
			"load15": load15,

			"uptime": uptime,
		}

		b, _ := json.Marshal(summary)
		hub.Broadcast <- b

		time.Sleep(10 * time.Second)
	}
}()
```

---

# 🎯 HASIL AKHIR

Kalau jalan:

## Browser akan menampilkan:

* CPU berubah realtime
* RAM berubah realtime
* Disk realtime
* Network live KB/s
* Load average live
* Uptime jalan
* Chart CPU bergerak

---

# 🚀 STATUS PROJECT SEKARANG

```text
███████████████████░░░ 85–90%
```

Kamu sudah punya:

✔ Monitoring engine
✔ SQLite storage
✔ API summary
✔ WebSocket realtime
✔ Dashboard UI
✔ Chart.js live
✔ Tailwind UI

---

# 🔥 NEXT STEP (FINAL POLISH)

Setelah ini kita bisa lanjut ke:

## 1. Gauge chart CPU / RAM / SSD (lebih keren)

## 2. Multi-chart network RX/TX

## 3. Export CSV

## 4. Systemd production hardening

## 5. Nginx HTTPS (domain pudin.eu.org)

---

Kalau dashboard sudah muncul di browser kamu, bilang:

👉 **“dashboard sudah jalan”**

nanti kita polish jadi level **Grafana mini versi kamu sendiri** 🔥
