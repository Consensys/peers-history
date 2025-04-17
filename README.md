A sidecar for Besu that keeps track of the peers history

It uses [admin_peers](https://besu.hyperledger.org/development/public-networks/reference/api#admin_peers) RPC API to fetch the peers connected to Besu and stores any changes in a SQLite database for long term persistance and also to a log file that be used for realtime analysis, for example using Loki to build Grafana dashboards.

The use of log files, instead of exposing metrics, is preferable in this case to avoid a label value explosion, since we are traking data with high cardinality like client versions, peer ids and ip addresses.

For each peer the following data is logged in a compact way to both reduce the size and simplify the querying via Loki:
```
10=besu⸱v24.8.0⸱linux-x86_64⸱openjdk-java-21⸱68⸱↓⸱99.159.194.247⸱16126⸱0x326c34de7c683cc2c11fb07c41659260435305926b7ae132e3adc00d0210d22adb3fd962fd1f298b1cc4d68e5faf11211b7431088fca795b3b5a407bdb4ab815
┃  ┃    ┃       ┃            ┃               ┃  ┃ ┃              ┃     ┗ peer id
┃  ┃    ┃       ┃            ┃               ┃  ┃ ┃              ┗ port
┃  ┃    ┃       ┃            ┃               ┃  ┃ ┗ ip
┃  ┃    ┃       ┃            ┃               ┃  ┗ direction (↓=inbound, ↑=outbound)
┃  ┃    ┃       ┃            ┃               ┗ eth protocol version
┃  ┃    ┃       ┃            ┗ runtime/lang
┃  ┃    ┃       ┗ os/arch
┃  ┃    ┗ client version
┃  ┗ client name
┗ connection slot
```
  
Usage:
```
peers-history -h
Usage: Peers History [-hvV] [-d=<logDir>] [-i=<scrapeIntervalSecs>]
[-l=<keepAliveLogIntervalSecs>] [-s=<dbDir>]
[-t=<ttlDays>] [-u=<rpcUri>]
-d, --log-dir=<logDir>   Where to put the logs. (default: .)
-h, --help               Show this help message and exit.
-i, --scrape-interval=<scrapeIntervalSecs>
Scrape interval in seconds. (default: 15)
-l, --keep-alive-log-interval=<keepAliveLogIntervalSecs>
Max interval between 2 logs. (default: 300)
-s, --db-dir=<dbDir>     Where to put the SQLite db. (default: .)
-t, --ttl=<ttlDays>      How many days of history to keep. (default: 365)
-u, --rpc-url=<rpcUri>   Execution engine RPC url to scrape. (default: http:
//localhost:8545)
-v, --verbose            Enable verbose output
-V, --version            Print version information and exit.
```


Sample Promtail configuration:
```
- job_name: peers-history
  pipeline_stages:
  - regex:
    expression: ^timestamp=(?P<timestamp>\d+),.*$
  - timestamp:
    format: Unix
    source: timestamp
  static_configs:
  - labels:
    __path__: /var/log/peers-history/peers-history.log.0
```