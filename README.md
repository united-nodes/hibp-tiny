# hibp-tiny

A stripped-down subset of the Have I Been Pwned (HIBP) password database, designed for offline auditing and low-memory environments.

The dataset includes only passwords that have been exposed more than 500 times in historical data breaches (based on the HIBP V8 dataset). It requires no database engine to query.

## Dataset Specifications

| Metric | HIBP Pwned Passwords V8 | hibp-tiny |
| :--- | :--- | :--- |
| **Criteria** | All exposed passwords | Breached > 500 times |
| **Total Records** | ~1.3 billion | 12 million |
| **Uncompressed Size** | ~37.3 GB | ~476 MB |

## Data Layout

To allow $O(1)$ lookups in plain text without memory overhead, the dataset is sharded and deduplicated:

*   **Sharding**: Hashes are split into 256 separate text files (`00.txt` through `FF.txt`) based on the first two hexadecimal characters of the SHA-1 hash.
*   **Prefix Stripping**: Because the filename implies the first two characters of the hash, these two characters are omitted from every line inside the file to save disk space (~24MB saved across the dataset).
*   **Count Removal**: Breach counts are removed. Only the remaining 38 characters of the SHA-1 hashes are stored.

The average size of a single shard is `~1.86 MB`.

## Usage

### Fetching Shards via CDN

Because individual shards are small, they can be fetched dynamically at runtime via public CDNs instead of cloning the entire repository. 

*   **jsDelivr**: `https://cdn.jsdelivr.net/gh/united-nodes/hibp-tiny@main/data/00.txt`
*   **Statically**: `https://cdn.statically.io/gh/united-nodes/hibp-tiny/main/data/00.txt`
*   **GitHub Raw**: `https://raw.githubusercontent.com/united-nodes/hibp-tiny/main/data/00.txt`

### Querying

To check a password against the dataset, you must hash it, extract the first two characters to locate the shard, and search the file for the remaining 38 characters.

Example using POSIX shell and standard utilities:

```bash
#!/bin/sh
# Usage: ./check.sh "password"

# Generate 40-character uppercase SHA-1 hash
HASH=$(printf "%s" "$1" | sha1sum | awk '{print toupper($1)}')

PREFIX=$(printf "%.2s" "$HASH")
SUFFIX=${HASH#??}

FILE="data/${PREFIX}.txt"

if [ ! -f "$FILE" ]; then
    echo "Error: $FILE not found."
    exit 1
fi

# Anchor search to line start (^) for fast evaluation
if grep -q "^$SUFFIX" "$FILE"; then
    echo "Password found in the high-risk dataset."
    exit 1
else
    echo "Password not found."
    exit 0
fi
```

## Note on Git History

This repository is used for data artifact distribution. To prevent repository bloat from large binary diffs, **Git history is intentionally discarded on every update**. The repository is maintained as a single orphan commit.

If you are cloning this repository locally, always use a shallow clone:

```bash
git clone --depth 1 https://github.com/united-nodes/hibp-tiny.git
```

## License

This project is in the public domain. See the [LICENSE](LICENSE) for details.
