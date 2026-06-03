# DynamoDB Leaderboard Data Modeling

## Requirement

Fields:

- user_id
- game_id
- score
- month (MM-YYYY)

Queries:

1. Update a user's score for a game/month.
2. Get the leaderboard (Top N users) for a game/month.

---

# Option 1: Write-Sharded Base Table + Leaderboard GSI

## Main Table: GameScores

### Primary Key

```text
PK = GAME#<game_id>#<month>#SHARD#<n>
SK = USER#<user_id>
```

Where:

```text
shard = hash(user_id) % 10
```

Example:

```text
PK = GAME#PUBG#01-2025#SHARD#3
SK = USER#123
```

### Item Example

```json
{
  "PK": "GAME#PUBG#01-2025#SHARD#3",
  "SK": "USER#123",
  "user_id": "123",
  "game_id": "PUBG",
  "month": "01-2025",
  "score": 9800,
  "shard": 3
}
```

---

## GSI1 (Leaderboard)

```text
GSI1PK = GAME#<game_id>#<month>
GSI1SK = SCORE#<inverted_score>
```

Because DynamoDB sorts ascending:

```text
inverted_score = 999999999 - score
```

Example:

```json
{
  "GSI1PK": "GAME#PUBG#01-2025",
  "GSI1SK": "SCORE#999990199"
}
```

---

# How Score Updates Work

Suppose user 123's score changes:

```text
9800 → 10500
```

### Step 1: Compute shard

```text
hash("123") % 10 = 3
```

### Step 2: Build key

```text
PK = GAME#PUBG#01-2025#SHARD#3
SK = USER#123
```

### Step 3: Update item

```sql
UpdateItem
PK = GAME#PUBG#01-2025#SHARD#3
SK = USER#123

SET score = 10500
```

DynamoDB automatically updates the GSI entry.

---

## Before Update

```json
{
  "PK": "GAME#PUBG#01-2025#SHARD#3",
  "SK": "USER#123",
  "score": 9800,
  "GSI1PK": "GAME#PUBG#01-2025",
  "GSI1SK": "SCORE#999990199"
}
```

## After Update

```json
{
  "PK": "GAME#PUBG#01-2025#SHARD#3",
  "SK": "USER#123",
  "score": 10500,
  "GSI1PK": "GAME#PUBG#01-2025",
  "GSI1SK": "SCORE#999989499"
}
```

DynamoDB removes the old GSI record and creates the new one automatically.

---

# Leaderboard Query

Query:

```text
GAME#PUBG#01-2025
```

on GSI1.

Result:

```text
User 101  Score 12000
User 220  Score 11900
User 333  Score 11800
...
```

Already sorted by score.

---

# Alternative Design (Usually Better)

If most operations are:

- Update user's score
- Fetch user's score

Use the user as the partition key.

## Main Table

```text
PK = USER#<user_id>
SK = GAME#<game_id>#<month>
```

Example:

```json
{
  "PK": "USER#123",
  "SK": "GAME#PUBG#01-2025",
  "score": 10500
}
```

Updating becomes trivial because the key is already known:

```text
PK = USER#123
SK = GAME#PUBG#01-2025
```

---

# Scaling Concern

When using:

```text
GSI1PK = GAME#PUBG#01-2025
```

every score update also updates the same GSI partition.

For very large games, the GSI becomes the hotspot.

---

# Write-Sharded GSI

Keep the main table:

```text
PK = USER#123
SK = GAME#PUBG#01-2025
```

But shard the leaderboard index:

```text
GSI1PK = GAME#PUBG#01-2025#SHARD#3
GSI1SK = SCORE#999989499
```

Where:

```text
shard = hash(user_id) % 10
```

---

# Reading the Leaderboard

Query:

```text
GAME#PUBG#01-2025#SHARD#0
GAME#PUBG#01-2025#SHARD#1
...
GAME#PUBG#01-2025#SHARD#9
```

Then:

1. Fetch top scores from each shard.
2. Merge-sort in the application.
3. Return the global Top 100.

---

# Recommendation

### Small / Medium Traffic

```text
Main Table:
PK = USER#user_id
SK = GAME#game_id#month

GSI:
PK = GAME#game_id#month
SK = SCORE#inverted_score
```

### Very High Traffic

```text
Main Table:
PK = USER#user_id
SK = GAME#game_id#month

Sharded GSI:
PK = GAME#game_id#month#SHARD#n
SK = SCORE#inverted_score
```

This is the typical large-scale leaderboard pattern used to avoid hot partitions while still supporting fast leaderboard queries.
