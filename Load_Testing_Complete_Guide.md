# Load Testing Guide
### Concepts, JMeter, and Modern Tools

---

## Table of Contents

1. What Is Load Testing (and Why It Matters)
2. Core Concepts Every Tester Must Know
3. Types of Performance Testing
4. Key Metrics You Will Be Judged On
5. The Load Testing Process (Step by Step)
6. Apache JMeter — Installation on Windows
7. Apache JMeter — Installation on macOS
8. Apache JMeter — Using It, End to End
9. Running JMeter Tests at Scale (Non-GUI & Distributed)
10. JMeter Plugins Worth Installing
11. Other Popular Load Testing Tools (Setup + Usage)
    - k6
    - Gatling
    - Locust
    - Apache Bench (ab) & wrk
    - Artillery
    - BlazeMeter
    - Postman/Newman for light load checks
12. Choosing the Right Tool for the Job
13. Reading & Interpreting Results
14. Common Mistakes Beginners Make
15. Best Practices Checklist
16. Glossary of Terms

---

## 1. What Is Load Testing (and Why It Matters)

Load testing is the practice of simulating many users (or requests) hitting a
system at the same time, to see how that system behaves under realistic or
extreme traffic. The goal is not to "break" the system for fun — it's to
answer business-critical questions before real users find the answers for
you:

- Can our checkout page survive Black Friday traffic?
- How many concurrent users can our API handle before response times become
  unacceptable?
- If traffic doubles overnight, does the system degrade gracefully or fall
  over completely?
- Where is the bottleneck — the app server, the database, the network, a
  third-party API?

Load testing sits inside the broader discipline of **performance testing**,
which also includes stress testing, soak testing, spike testing, and
scalability testing (all explained in Section 3).

Without load testing, teams find out about capacity problems in production —
usually during the worst possible moment (a big sale, a marketing campaign,
a product launch).

---

## 2. Core Concepts Every Tester Must Know

| Term | Plain-English Meaning |
|---|---|
| **Virtual User (VU) / Thread** | A simulated user. If you run 100 threads in JMeter, you are simulating 100 people using the app at once. |
| **Concurrency** | How many virtual users are active at the exact same moment. |
| **Throughput** | How many requests the system processes per unit of time (usually requests/second or transactions/second). |
| **Response Time / Latency** | How long it takes for the system to respond to a single request. |
| **Ramp-up Period** | The time taken to bring all virtual users online gradually, instead of firing them all at once (which is unrealistic and can itself crash a server). |
| **Ramp-down** | Gradually stopping virtual users at the end of a test. |
| **Think Time** | A pause between actions to mimic a real human reading a page, filling a form, etc., instead of hammering the server non-stop. |
| **Steady State** | The period during a test where load stays flat, letting you observe stable behavior. |
| **Pacing** | Controlling how often a virtual user repeats a scenario, to hit a target throughput precisely. |
| **SLA (Service Level Agreement)** | The agreed performance target, e.g. "95% of requests must respond in under 2 seconds." |
| **Percentiles (p90, p95, p99)** | A better way to look at response time than "average." p95 = 2s means 95% of requests were faster than 2 seconds — this reveals the experience of your slowest users, which averages hide. |
| **Error Rate** | The percentage of requests that failed (timeouts, 5xx errors, etc.) during the test. |
| **Bottleneck** | The single component (CPU, memory, DB connections, network bandwidth, thread pool, etc.) that limits how much load the system can take. |
| **Correlation** | Capturing a dynamic value from one response (like a session token or CSRF token) and reusing it in the next request — essential for realistic scripts. |
| **Parameterization** | Feeding different data into each virtual user (different usernames, product IDs, etc.) instead of everyone using the same static value. |

---

## 3. Types of Performance Testing

Load testing is one branch of a bigger tree. Know the difference — interviewers
and tickets will use these terms precisely.

- **Load Testing** — Simulate expected (or slightly above expected) real-world
  traffic to see how the system performs under normal-to-peak conditions.
- **Stress Testing** — Push load far beyond expected capacity until the system
  breaks, to find the breaking point and see *how* it fails (gracefully with
  proper error messages, or catastrophically).
- **Spike Testing** — Suddenly throw a huge burst of traffic at the system
  (e.g., go from 50 to 5,000 users in seconds) to see how it handles sudden
  surges, like a flash sale link going viral.
- **Soak / Endurance Testing** — Run a moderate, realistic load for a long
  duration (several hours to days) to catch problems that only appear over
  time: memory leaks, disk filling up with logs, connection pools not being
  released, gradual performance decay.
- **Scalability Testing** — Gradually increase load in stages while adding
  resources, to find out whether the system scales linearly (2x resources =
  2x capacity) or hits diminishing returns.
- **Volume/Flood Testing** — Focus on huge amounts of *data* rather than
  users (e.g., a database with 50 million rows) to test how the system copes.

---

## 4. Key Metrics You Will Be Judged On

When you present load test results, these are the numbers stakeholders
actually care about:

1. **Response Time (avg, median, p90, p95, p99, max)**
2. **Throughput** (requests/sec or transactions/sec)
3. **Error Rate** (%)
4. **Concurrent Users Supported** at an acceptable response time/error rate
5. **Server-side resource usage** during the test — CPU %, memory, disk I/O,
   network I/O, DB query time, connection pool usage (usually pulled from
   monitoring tools like Grafana, Datadog, New Relic, or cloud provider
   dashboards, not from the load tool itself)
6. **Time to first byte (TTFB)** for web apps
7. **Saturation point** — the load level at which response time starts
   increasing sharply (the "knee" of the curve)

A good report always pairs client-side numbers (what the load tool measured)
with server-side numbers (what the infrastructure was doing), because a
slow response time alone doesn't tell you *why* it was slow.

---

## 5. The Load Testing Process (Step by Step)

1. **Define objectives** — What question are you answering? ("Can we support
   2,000 concurrent users at checkout with p95 < 3s and error rate < 1%?")
2. **Identify critical scenarios** — Usually the top 20% of user journeys that
   cause 80% of load: login, search, add-to-cart, checkout, an API's hottest
   endpoint.
3. **Gather requirements/NFRs** — Expected user counts, peak hours, growth
   projections, existing SLAs.
4. **Design the test scenario** — Decide the mix of actions (e.g. 40% browse,
   30% search, 20% add-to-cart, 10% checkout), think times, and data needs.
5. **Prepare test data** — Unique usernames, product IDs, payment tokens,
   etc., so you're not hammering the same row/user (unrealistic and can
   cause false locking bottlenecks).
6. **Set up environment** — Ideally a production-like (staging) environment.
   Testing load against a tiny dev box gives meaningless numbers. Also make
   sure the environment is isolated so you don't accidentally DDoS a shared
   or production system.
7. **Script the test** — Build it in JMeter/k6/Gatling/etc. Record or code
   the requests, then add correlation, parameterization, and assertions.
8. **Do a dry run** — Run with 1–5 users first to confirm the script works
   correctly and the assertions pass, before scaling up.
9. **Execute the real test(s)** — Load, stress, spike, soak, as planned.
10. **Monitor everything while it runs** — Client-side (the tool) and
    server-side (APM/monitoring) at the same time.
11. **Analyze results** — Look at percentiles, error rate, throughput vs.
    concurrency graphs, resource graphs, and correlate them to find the
    actual bottleneck.
12. **Report and recommend** — Summarize findings for both technical and
    non-technical audiences; recommend fixes or scaling changes.
13. **Retest after fixes** — Performance tuning is iterative.

---

## 6. Apache JMeter — Installation on Windows

JMeter is a free, open-source Java application, so Java must be installed
first.

### Step 1: Install Java (JDK)
1. Download a JDK (Java 8 or newer; Java 11/17 LTS recommended) from
   Adoptium (Eclipse Temurin) or Oracle.
2. Run the installer and complete setup with default options.
3. Verify installation by opening **Command Prompt** and typing:
   ```
   java -version
   ```
   You should see a version number printed back.
4. Set the `JAVA_HOME` environment variable if it isn't set automatically:
   - Search "Environment Variables" in the Start Menu → Edit the system
     environment variables → Environment Variables.
   - Under **System variables**, click **New** → Variable name: `JAVA_HOME`,
     Variable value: the JDK install path (e.g.
     `C:\Program Files\Eclipse Adoptium\jdk-17`).
   - Edit the `Path` variable and add `%JAVA_HOME%\bin`.

### Step 2: Download JMeter
1. Go to the official Apache JMeter downloads page
   (jmeter.apache.org → Download).
2. Download the **Binary** `.zip` (not the source archive).

### Step 3: Extract and Run
1. Extract the `.zip` file to a simple path, e.g. `C:\apache-jmeter`.
2. Open the extracted folder → go into the `bin` folder.
3. Double-click `jmeter.bat` to launch the JMeter GUI.
   (Alternatively, run it from Command Prompt: `jmeter.bat`)
4. The JMeter GUI window should open within a few seconds.

### Optional: Add JMeter to PATH
So you can type `jmeter` from any folder in Command Prompt:
1. Copy the full path to the `bin` folder (e.g. `C:\apache-jmeter\bin`).
2. Environment Variables → edit `Path` → add that folder.
3. Restart Command Prompt and test with `jmeter -v`.

---

## 7. Apache JMeter — Installation on macOS

### Option A: Using Homebrew (recommended, fastest)
1. Install Homebrew if you don't already have it (from brew.sh — run the
   install command in Terminal).
2. Install Java if needed:
   ```
   brew install openjdk@17
   ```
   Follow the on-screen instructions Homebrew prints to link it (it usually
   asks you to add a symlink so macOS's Java wrappers can find it).
3. Install JMeter:
   ```
   brew install jmeter
   ```
4. Launch it:
   ```
   jmeter
   ```
   That's it — Homebrew handles PATH setup automatically.

### Option B: Manual Installation
1. Install a JDK (Adoptium Temurin `.pkg` installer is the easiest for Mac).
2. Verify with:
   ```
   java -version
   ```
3. Download the JMeter binary `.zip` (better than `.tgz` if you're unsure how
   to extract tarballs from Finder) from jmeter.apache.org.
4. Extract it (double-click in Finder, or `unzip filename.zip` in Terminal)
   to a folder like `/Applications/apache-jmeter`.
5. Open Terminal and navigate to the `bin` folder:
   ```
   cd /Applications/apache-jmeter/bin
   ./jmeter
   ```
6. (Optional) Add JMeter's `bin` folder to your PATH by adding this line to
   `~/.zshrc` (or `~/.bash_profile` on older Macs):
   ```
   export PATH="/Applications/apache-jmeter/bin:$PATH"
   ```
   Then run `source ~/.zshrc` and test with `jmeter -v` from anywhere.

**Note (Apple Silicon M1/M2/M3):** JMeter runs fine on Apple Silicon since
it runs on the JVM; just make sure you install an ARM-native JDK (Homebrew
handles this automatically).

---

## 8. Apache JMeter — Using It, End to End

JMeter's GUI is used to **build and debug** test plans. For actually
**running** big load tests, you switch to non-GUI/command-line mode
(covered in Section 9) — running heavy load from the GUI itself will give
inaccurate results because the GUI consumes CPU/memory too.

### 8.1 Understand the Test Plan Structure (Tree)

Everything in JMeter lives inside a **Test Plan**, structured as a tree:

```
Test Plan
 └── Thread Group                  (defines virtual users)
      ├── HTTP Request Defaults    (config element - optional but recommended)
      ├── HTTP Cookie Manager      (config element - handles sessions/cookies)
      ├── HTTP Header Manager      (config element - sets common headers)
      ├── HTTP Request (Sampler)   (the actual request, e.g. GET /login)
      │    └── Response Assertion  (checks the response is correct)
      ├── HTTP Request (Sampler)   (next request, e.g. POST /cart)
      ├── Timer                    (adds think time between requests)
      └── Listener                 (View Results Tree, Summary Report, etc.)
```

### 8.2 Build Your First Test Plan

1. Open JMeter. In the left tree, right-click **Test Plan** → **Add** →
   **Threads (Users)** → **Thread Group**.
2. In the Thread Group panel, set:
   - **Number of Threads (users)** — e.g. 50 (how many virtual users)
   - **Ramp-up period (seconds)** — e.g. 30 (spread user start over 30s)
   - **Loop Count** — how many times each user repeats the scenario (or check
     "Infinite" and control duration separately with a Duration/Scheduler
     setting)
3. Right-click the Thread Group → **Add** → **Sampler** → **HTTP Request**.
   - Set **Server Name or IP** (e.g. `example.com`)
   - Set **Path** (e.g. `/api/products`)
   - Choose **Method** (GET, POST, etc.)
   - For POST requests, add body data or parameters in the "Body Data" or
     "Parameters" tab.
4. Right-click Thread Group → **Add** → **Listener** → **View Results Tree**
   (great for debugging — shows the actual request/response for each call).
5. Also add **Summary Report** and/or **Aggregate Report** listeners — these
   show you the real performance numbers (avg, min, max, throughput, error %).
6. Click the green **Start** (play) button in the toolbar to run.
7. Watch results populate in your listeners in real time.

### 8.3 Making It Realistic

- **HTTP Cookie Manager**: Add this so JMeter handles session
  cookies automatically, like a real browser.
- **HTTP Header Manager**: Add common headers (Content-Type,
  Authorization, User-Agent) so requests look like they come from a real
  client/app rather than a bot.
- **Timers** (e.g. "Uniform Random Timer" or "Constant Timer"): Add between
  requests to simulate human think time. Right-click a sampler → Add →
  Timer.
- **CSV Data Set Config**: Right-click Thread Group → Add → Config Element →
  CSV Data Set Config. Point it at a `.csv` file (e.g. usernames/passwords)
  so each virtual user uses different data instead of hammering with one
  hardcoded value.
- **Correlation (extracting dynamic values)**: Use a **Regular Expression
  Extractor** or **JSON Extractor** (right-click a sampler → Add →
  Post Processor) to pull a value out of a response (like a session token or
  CSRF token) and store it in a variable, then reference that variable
  (`${variableName}`) in the next request. This is essential — without it,
  multi-step flows like login → checkout will fail because the app expects a
  fresh token each time.
- **Assertions**: Right-click a sampler → Add → Assertions → Response
  Assertion. Set it to check that the response contains expected text or
  returns the right status code, so JMeter flags real failures, not just
  "got a response."

### 8.4 Recording a Script from Real Browser Actions

Instead of building every request manually, you can record real clicks:

1. Right-click **Test Plan** → Add → Non-Test Elements → **HTTP(S) Test
   Script Recorder**.
2. Set the recorder's port (default 8888) and target controller (usually a
   Thread Group you've created).
3. Under the recorder's "Add suggested Excludes" button, exclude static
   assets (`.css`, `.js`, `.png`, etc.) so you don't record noise.
4. Click **Start** on the recorder.
5. Configure your browser to use `localhost:8888` as its proxy (or install
   the JMeter root CA certificate if the site uses HTTPS — the recorder's
   panel has a button to generate/trust this certificate).
6. Browse the site normally through that proxied browser — every request
   gets recorded into your Thread Group.
7. Click **Stop** on the recorder when done, then clean up the recorded
   requests (remove duplicates/unwanted calls, add correlation/assertions).

### 8.5 Common Elements Cheat-Sheet

| Element | Purpose |
|---|---|
| Thread Group | Defines number of users, ramp-up, loops |
| HTTP Request | A single API/page call |
| HTTP Request Defaults | Set server/domain once for all requests |
| HTTP Cookie/Header Manager | Session + header handling |
| CSV Data Set Config | Feed different data to each user |
| Regular Expression / JSON Extractor | Correlation — capture dynamic values |
| Response Assertion | Pass/fail validation of a response |
| Timers | Think time / pacing |
| Listeners (View Results Tree, Summary Report, Aggregate Report) | See results — use View Results Tree only for debugging with few users |
| Controllers (Loop, If, Transaction) | Control test flow/logic |

---

## 9. Running JMeter Tests at Scale (Non-GUI & Distributed)

**Never run a real load test with hundreds/thousands of users through the
GUI.** The GUI itself consumes resources and skews results. Build your
script in the GUI, then run it headless.

### 9.1 Non-GUI (CLI) Mode
```
jmeter -n -t test_plan.jmx -l results.jtl -e -o report_output_folder
```
- `-n` → run in non-GUI mode
- `-t` → path to your `.jmx` test plan file
- `-l` → where to save raw results (a `.jtl` file)
- `-e -o` → automatically generate an HTML dashboard report from the results
  into the given folder after the run finishes

Open `report_output_folder/index.html` afterward for graphs, percentiles,
throughput charts, and error breakdowns — much nicer than reading raw
listener output.

### 9.2 Distributed (Multi-Machine) Testing

If one machine can't generate enough load (its own CPU/network becomes the
bottleneck before your target system's does), spread the load across
multiple machines:

1. Install the same JMeter version on all machines.
2. On each **worker/slave** machine, start the JMeter server process:
   ```
   jmeter-server
   ```
3. On the **controller/master** machine, edit `jmeter.properties` (or pass
   via CLI) to list the worker IPs, then run:
   ```
   jmeter -n -t test_plan.jmx -R ip1,ip2,ip3 -l results.jtl
   ```
4. The master coordinates the workers and aggregates results into one file.

Cloud-based alternatives (BlazeMeter, Azure Load Testing, AWS Distributed
Load Testing solution) exist specifically to avoid manually managing worker
machines.

---

## 10. JMeter Plugins Worth Installing

Install the **JMeter Plugins Manager** (a `.jar` you drop into JMeter's
`lib/ext` folder from jmeter-plugins.org), which then gives you a GUI
"Plugins Manager" menu inside JMeter to install more:

- **Custom Thread Groups** (Ultimate Thread Group, Stepping Thread Group) —
  build complex, staged load patterns visually instead of only linear
  ramp-ups.
- **PerfMon (Server Agent)** — collects CPU/memory/disk stats from your
  servers during the test and overlays them on JMeter's graphs.
- **Throughput Shaping Timer** — precisely control requests/second over
  time (great for spike tests).
- **3 Basic Graphs / Response Times Over Time** — nicer real-time graphing
  than the default listeners.

---

## 11. Other Popular Load Testing Tools

JMeter isn't the only game in town. Here's what's widely used today, how
they differ, and how to get started with each.

### k6 (by Grafana Labs)
Modern, developer-first, scripted in **JavaScript**. Very popular for
CI/CD pipelines because it's a single lightweight binary (no JVM needed)
and results integrate nicely with Grafana dashboards.

**Install:**
- macOS: `brew install k6`
- Windows: `choco install k6` (via Chocolatey) or `winget install k6
  --source winget`, or download the `.msi` from k6.io

**Basic usage** — save a script as `test.js`:
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  vus: 50,            // virtual users
  duration: '2m',      // test duration
};

export default function () {
  const res = http.get('https://example.com/api/products');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);            // think time
}
```
Run it:
```
k6 run test.js
```
For staged ramp-up/ramp-down instead of flat VUs, use `options.stages`
(an array of `{ duration, target }` objects) instead of a flat `vus`.

### Gatling
Scala-based (but has a no-code recorder), known for excellent HTML reports
and being resource-efficient for high loads from a single machine.

**Install:** Download the open-source bundle from gatling.io, unzip, and
run `bin/gatling.sh` (Mac/Linux) or `bin\gatling.bat` (Windows). Requires
Java, same as JMeter.

**Basic usage:** Gatling has a built-in **Recorder** (similar to JMeter's)
to capture browser traffic and generate a simulation script in Scala or
Java automatically. You then run:
```
./gatling.sh
```
and select the simulation to execute; an HTML report is generated
automatically after each run.

### Locust
Python-based, open-source, popular with teams that already write Python
and want load scenarios as plain Python code instead of a GUI/DSL.

**Install:**
```
pip install locust
```

**Basic usage** — save as `locustfile.py`:
```python
from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    wait_time = between(1, 3)  # think time between tasks

    @task
    def load_homepage(self):
        self.client.get("/")

    @task(3)  # 3x more likely to run than load_homepage
    def view_product(self):
        self.client.get("/products/123")
```
Run it:
```
locust -f locustfile.py
```
Then open `http://localhost:8089` in a browser — Locust gives you a live
web UI where you type in the number of users and spawn rate, click Start,
and watch real-time charts.

### Apache Bench (ab) and wrk
Lightweight command-line tools for quick, simple benchmarks of a single URL
— not for complex multi-step scenarios, but great for a fast sanity check.

**Apache Bench** (often pre-installed with Apache, or `apt install
apache2-utils` / `brew install httpd`):
```
ab -n 1000 -c 50 https://example.com/
```
(`-n` = total requests, `-c` = concurrency)

**wrk** (`brew install wrk` on Mac; build from source or use WSL on
Windows since native Windows support is limited):
```
wrk -t4 -c100 -d30s https://example.com/
```
(`-t` = threads, `-c` = connections, `-d` = duration)

### Artillery
Node.js-based, config-driven (YAML or JS), popular for API and WebSocket
load testing, integrates well into CI/CD.

**Install:**
```
npm install -g artillery
```
**Basic usage** — save as `test.yml`:
```yaml
config:
  target: "https://example.com"
  phases:
    - duration: 60
      arrivalRate: 10
scenarios:
  - flow:
      - get:
          url: "/api/products"
```
Run it:
```
artillery run test.yml
```

### BlazeMeter
A commercial (with free tier) cloud platform built on top of JMeter and
other open-source engines. You upload your existing `.jmx` file (or build
scriptlessly), and it runs the load from BlazeMeter's cloud infrastructure
across multiple geographic regions, then gives you polished dashboards,
trend reports, and team collaboration features — useful when you need huge
scale without managing your own worker machines, or need results in a
format non-technical stakeholders can read easily.

### Postman / Newman
Not a dedicated load tool, but often used for **light** load/smoke checks
since many teams already have Postman collections. **Newman** is Postman's
CLI runner; combined with a loop or a tool like `newman-reporter` /
`loadtest` npm packages, it can fire moderate concurrent requests. For
serious load testing, prefer JMeter/k6/Gatling/Locust — Postman/Newman
doesn't scale to thousands of concurrent virtual users cleanly.

---

## 12. Choosing the Right Tool for the Job

| Situation | Good Fit |
|---|---|
| You're new to load testing, want a GUI, need broad protocol support (HTTP, JDBC, FTP, JMS, SOAP, etc.) | **JMeter** |
| You/your team is already in JavaScript, want CI/CD-native, lightweight | **k6** |
| You need very high throughput from limited hardware, love clean HTML reports | **Gatling** |
| Your team is Python-first, wants readable scripts-as-code | **Locust** |
| You just need a 30-second sanity check on one endpoint | **ab** or **wrk** |
| You want config-driven YAML tests for APIs/WebSockets in a Node pipeline | **Artillery** |
| You need massive distributed scale without managing infrastructure, or need shareable dashboards for non-engineers | **BlazeMeter** (or similar cloud service) |

---

## 13. Reading & Interpreting Results

- **Look at percentiles, not just averages.** An average of 500ms can hide
  the fact that 5% of users are waiting 8 seconds. Always check p90/p95/p99.
- **Watch the shape of the throughput-vs-response-time curve.** Response
  time should stay roughly flat as load increases — until it hits a
  bottleneck, where it starts climbing sharply ("the knee"). That knee is
  your system's practical capacity limit.
- **Correlate client-side and server-side data.** If response times spike
  and, at the same moment, DB CPU maxes out, you've found your bottleneck.
  If the load tool's own machine is maxed on CPU/network, your results are
  invalid — you're bottlenecked on the load generator, not the system under
  test.
- **Check the error rate trend, not just the total.** Errors that start
  appearing only after a certain concurrency level pinpoint the exact
  breaking point.
- **Sanity check the results.** If throughput mysteriously plateaus while
  CPU/memory on the server stays low, suspect the load generator itself,
  network limits, or a connection-pool/thread-limit setting client-side.

---

## 14. Common Mistakes Beginners Make

- Running heavy load through the JMeter **GUI** instead of CLI/non-GUI mode
  (skews results because the GUI itself eats CPU/RAM).
- Using the **same test data** for every virtual user (causes artificial
  locking/caching effects that don't reflect real usage).
- Not adding **think time**, so the test simulates unrealistic
  "robot-speed" hammering instead of real user behavior.
- Ramping up users **instantly** instead of gradually (unrealistic and can
  crash a server that would have handled the same load fine if it arrived
  gradually).
- Forgetting to add **assertions**, so the test reports "100% success"
  even when the server is silently returning error pages with a 200 status.
- Testing against a tiny, underpowered, or misconfigured environment and
  assuming results predict production behavior.
- Not correlating dynamic values (session tokens, CSRF tokens), so
  multi-step flows silently fail after the first request.
- Ignoring the **network** between the load generator and the target — if
  you're testing a system across the public internet, the load generator's
  bandwidth might be the real bottleneck.
- Only measuring average response time and missing tail latency (p95/p99)
  that real users actually experience.
- Not monitoring the **server side** at all — only seeing "it got slow"
  without knowing *why*.

---

## 15. Best Practices Checklist

- [ ] Define clear, measurable objectives (target users, target response
      time, acceptable error rate) before writing a single script.
- [ ] Test against a production-like environment, isolated from real users.
- [ ] Use realistic, varied test data (CSV Data Set Config, parameterized
      scripts).
- [ ] Add think time and realistic ramp-up/ramp-down.
- [ ] Add assertions so failures are actually detected.
- [ ] Run scripts in CLI/non-GUI mode for real load; use GUI only for
      building/debugging with a handful of users.
- [ ] Monitor server-side resources (CPU, memory, DB, network) during every
      run, alongside the load tool's own metrics.
- [ ] Start small (a dry run with 1–5 users), verify correctness, then
      scale up.
- [ ] Automate load tests in CI/CD for regression detection over time
      (k6 and Artillery are especially popular for this).
- [ ] Document and version-control your test scripts like any other code.
- [ ] Re-test after every significant fix or infrastructure change.
- [ ] Report results with percentiles and graphs, not just a single
      average number.

---

## 16. Glossary of Terms

- **APM** — Application Performance Monitoring (e.g., New Relic, Datadog,
  Dynatrace) — tools that show what's happening inside the server during a
  test.
- **CI/CD** — Continuous Integration / Continuous Deployment — automated
  pipelines that can trigger load tests automatically on new builds.
- **DSL** — Domain-Specific Language — a mini-language for writing tests
  (Gatling and k6 both effectively provide one).
- **JMX file** — JMeter's native test plan file format (XML-based).
- **JTL file** — JMeter's raw results log file.
- **SLA/NFR** — Service Level Agreement / Non-Functional Requirement —
  the agreed performance target the test is measuring against.
- **Thread Pool / Connection Pool** — A limited set of workers/DB
  connections a server reuses; exhausting this is one of the most common
  real-world bottlenecks found by load testing.
- **VU (Virtual User)** — A simulated user generated by the load tool.

---

*Samina Afroze*
