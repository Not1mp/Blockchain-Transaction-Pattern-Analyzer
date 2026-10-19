# Blockchain-Transaction-Pattern-Analyzer

A PySpark project that looks at how money moves between Ethereum wallets and flags wallets with suspicious patterns.

It reads about 4.9 million transactions, turns them into a graph (wallets are dots, transfers are lines between them), and gives every wallet a risk level: **Low**, **Medium** or **High**.

## How it works

1. **Load the data.** Read 50 Parquet files of Ethereum transactions from Google Cloud Storage.
2. **Build the graph.** Every unique wallet address becomes a node. Every transaction becomes an edge from the sender to the receiver (using GraphFrames).
3. **Create features.** Calculate 14 numbers that describe each wallet's behaviour.
4. **Score the risk.** Apply 7 simple rules. Each rule adds points, and the total decides the risk level.
5. **Save the results.** Write one CSV with every wallet, its features and its risk level.

## The data

- Source: one month of transactions from Google's public Ethereum dataset on BigQuery, exported as Parquet files.
- Location used in the notebook: `gs://blockchain_transac_data/jan_subgraph_filtered`
- Size: **4,943,153 transactions** and **2,361,095 unique wallets**.
- Main columns used: `from_address`, `to_address`, `value`, `block_timestamp`, `hash`.

The data is not included in this project. To use your own, export the Ethereum transactions table from BigQuery as Parquet and point the notebook to it.

## Wallet features

| Feature | Meaning |
|---------|---------|
| `degree` | Total number of transactions (sent + received) |
| `inDegree` | Number of transactions received |
| `outDegree` | Number of transactions sent |
| `unique_receivers` | How many different wallets it sent money to |
| `unique_senders` | How many different wallets it received money from |
| `total_counterparties` | `unique_receivers` + `unique_senders` |
| `total_sent` / `total_received` | Total value sent / received |
| `avg_sent` / `avg_received` | Average value per transaction sent / received |
| `active_days` | Number of days the wallet sent transactions |
| `avg_daily_tx` | `degree` divided by active days |
| `fan_out_ratio` | Sent transactions compared to received (high = spreads money out) |
| `fan_in_ratio` | Received transactions compared to sent (high = gathers money in) |

## Risk rules

A wallet gets points for each rule it meets:

| Rule | Points |
|------|:------:|
| More than 500 transactions | +2 |
| More than 2,000 transactions | +3 |
| More than 500 incoming transactions | +2 |
| More than 500 outgoing transactions | +2 |
| Fan-out ratio above 10 | +2 |
| Fan-in ratio above 10 | +2 |
| Active on more than 20 days | +1 |

**Risk level:** 6 or more points = **High**, 3 to 5 points = **Medium**, below 3 = **Low**.

The notebook also looks for three suspicious patterns:

- **Hubs:** wallets with a very large number of transactions.
- **Splitters:** wallets that send to many wallets but receive from few (more than 500 out, fewer than 100 in).
- **Collectors:** wallets that receive from many wallets but send to few (more than 500 in, fewer than 100 out).

## Results

| Risk level | Wallets |
|------------|--------:|
| High | 605 |
| Medium | 835 |
| Low | 2,359,655 |

Only **1,440 wallets (0.06%)** were flagged as Medium or High.

## How to run

The notebook is written for **Google Colab**, since the file paths start with `/content/`.

1. Upload `Blockchain_graph_Analysis_code__1_.ipynb` to Google Colab.
2. Run the cells from top to bottom. They will:
   - Copy the Parquet files from Google Cloud Storage
   - Install Java 17 and Apache Spark 3.5.0
   - Install `findspark` and `graphframes`
   - Build the graph, create the features and score the wallets
3. Find the results in `/content/final_wallet_features_csv`. Download this folder before the Colab session ends.

Spark is set to use 8 GB of memory, so choose a Colab runtime with enough RAM.

## Good to know

- The risk score is a simple rule-based check, not proof of wrongdoing. Busy wallets such as exchanges, bots and smart contracts can also score high.
- The thresholds (500, 2,000, 10 and so on) were chosen by hand and can be tuned.
- `active_days` only counts days when the wallet sent transactions.
- The results cover one month of data, so long-term behaviour is not captured.
