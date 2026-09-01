# TiDB LATAM Hackathon 2026

Main repository for the TiDB LATAM Hackathon. Build something with **TiDB Cloud**, **AWS**, and an **LLM API**.

**Event ends 2026-09-05.** All AWS accounts, EC2 instances, S3 data, TiDB clusters, and API keys are deleted after that date.

**Two regions, on purpose.** The AWS console and your EC2 instance are in `sa-east-1` (São Paulo). Bedrock is in `ap-southeast-1` (Singapore). That split is expected, not a bug — point your Bedrock client at Singapore and everything works.

**Nothing is done in advance.** Teams are formed on the morning of the event from the confirmed attendee list and announced at the venue. Everything else — your AWS account, your Bedrock key, installing Kiro, your TiDB cluster — happens on site, in a guided setup block before the 2h30 build clock starts.

---

## Start Here

📄 **[Ask the Airport.pdf](Ask%20the%20Airport.pdf)** — the official event brief: format, scoring breakdown, suggested directions, and the 2h30 timeline.

👉 **[PARTICIPANT-GUIDE.md](PARTICIPANT-GUIDE.md)** — sign-in, MFA, EC2, S3, TiDB, network limits, and how to submit.

Read both before asking an organizer anything. Most "permission denied" messages are documented in the guide and are intentional.

---

## The Challenge

Build a generative AI application in one afternoon. **That is the whole brief** — we are not prescribing the problem.

| | |
|---|---|
| **Date** | Wed 02 Sep 2026, São Paulo |
| **Build sprint** | 2h30 |
| **Teams** | 10 |
| **Team size** | 4–5 people |
| **Demo** | 2 minutes max |
| **Total points** | 100 |

Bring a perspective: an assistant, an analytics tool, a copilot, an agent, something we haven't thought of. Judging rewards **clever ideas and genuine business value** far more than it rewards checking boxes.

---

## The Dataset — `airportdb`

We have prepared a curated airline database. You do not have to use it, but it is there because it is **rich in interesting problems**: delays cascade, connections break, weather disrupts, passengers have preferences and histories.

Teams that find a real problem in this data and solve it well will score higher on creativity and business value than teams starting from a blank sheet.

| | |
|---|---|
| **Window** | 2015-06-02 to 2015-06-09 |
| **Scope** | All flights touching a Brazilian airport, plus a global sample for variety |
| **Size** | 5,191 flights · 617,062 bookings · 36,095 passengers · weather included |
| **Tables** | 12 — `airline` `airplane` `airplane_type` `airport` `airport_geo` `booking` `employee` `flight` `flightschedule` `passenger` `passengerdetails` `weatherdata` |
| **Download** | <https://hackaton-tidb.s3.sa-east-1.amazonaws.com/dumps/hackathon_airportdb.sql.gz> — 10 MB gzipped, public |

```bash
curl -O https://hackaton-tidb.s3.sa-east-1.amazonaws.com/dumps/hackathon_airportdb.sql.gz
gunzip hackathon_airportdb.sql.gz
mysql -h <tidb-host> -P 4000 -u <user> -p --ssl-ca=/etc/ssl/cert.pem airportdb < hackathon_airportdb.sql
```

TLS is required on the TiDB public endpoint. If the venue wifi makes the import crawl, use **TiDB Cloud's import-from-S3** in the console instead — the cluster fetches it server-side.

Row counts above are taken from the dump itself; the event brief's "8 tables, ~500K bookings" understates it.

### Dig for the problem first

Before writing code, spend a few minutes **querying the data and looking for what is broken in it**. The interesting projects come from a real pattern someone noticed, not from a feature list.

Questions worth asking the data:

- Which connections break most often, and does weather explain it?
- Which routes are chronically late, and who is sitting on those planes?
- What does a passenger's booking history say about what they would accept as a rebooking?
- Where does the schedule promise something the aircraft rotation cannot deliver?

### Directions worth stealing

- **Natural-language analytics** — a question in, SQL generated, answer out.
- **Disruption copilot** — cross-reference flights and weather to predict which connections break, and explain why in plain language.
- **Travel agent with memory** — learn a passenger's preferences in one conversation, apply them in the next. Rebook them when their flight slips three hours.
- **Semantic concierge** — vector search over routes and notes for "find me something like this".
- **Something else entirely** — genuinely, the creativity points are real.

> **If you ask Kiro to pick the problem, it will pick the same one as the other nine teams. Choosing the problem is yours to make.**

---

## Expected Stack

You are free to build however you like, but these are what the event provides and what the scoring rewards.

### Kiro — AI development tool

We expect teams to build with **[Kiro](https://kiro.dev)**. **Install it yourself before the event** — it is not handed out on site, and downloading an IDE over venue wifi will cost you part of your sprint. Commit your `.kiro/` specs to your repository — they are read during judging and are among the cheapest points on the board.

**Suggested way to work:** put **one laptop on the screen and let the AI drive the development and deployment**, while the whole team watches, argues, and steers. Two and a half hours is not enough time for four people to write code in parallel and merge it. It *is* enough time for four people to out-think a problem together while one machine does the typing.

Assign roles instead of splitting the codebase: **data · AI · interface · pitch**.

### AWS EC2

One instance per team, in **sa-east-1 (São Paulo)**, tagged with your team's username. Connect via Session Manager — no SSH key, no open port 22. See the [participant guide](PARTICIPANT-GUIDE.md).

**Ports 8000–8999 and 3000 are open inbound**, so a web app you run there is reachable at `http://<instance-public-ip>:8501`. Bind to `0.0.0.0`, not `127.0.0.1`. The public IP changes on every stop/start, and anything you serve is public — no auth, no TLS.

Deploying there is worth points, but do not let it eat your submission window. Running locally and recording the demo beats missing the deadline.

### Amazon Bedrock

Your Bedrock API key is issued for **ap-southeast-1 (Singapore)** — not São Paulo, where the console and your EC2 instance live. That is deliberate. Point your client at Singapore; the key will not work anywhere else. A client left on `sa-east-1` fails with an **authentication / access-denied error**, not a "model not found" — so when you see "denied", check the region before anything else.

Available on-demand models there:

| Purpose | Model ID |
|---|---|
| Text | `anthropic.claude-3-5-sonnet-20240620-v1:0` |
| Text (faster, cheaper) | `anthropic.claude-3-haiku-20240307-v1:0` |
| Embeddings | `cohere.embed-english-v3` — 1024 dimensions |
| Embeddings (multilingual) | `cohere.embed-multilingual-v3` — 1024 dimensions |

`amazon.titan-embed-text-v2:0` and the Mistral models are **not** available in ap-southeast-1. If you want Titan embeddings, use TiDB's built-in `EMBED_TEXT()` instead — it does not go through Bedrock at all, needs no key, and is the shortest path to working vector search.

### Using the key

Copy these two blocks as they are — the region is already correct. Put the key in your config file, never in a commit:

```bash
# .env  —  git-ignored
AWS_BEARER_TOKEN_BEDROCK=<the key handed to your team on site>
AWS_REGION=ap-southeast-1
```

```python
import json, os, boto3
from dotenv import load_dotenv

load_dotenv()  # exports AWS_BEARER_TOKEN_BEDROCK into the environment

bedrock = boto3.client("bedrock-runtime", region_name="ap-southeast-1")  # Singapore, not São Paulo

def ask(prompt: str) -> str:
    resp = bedrock.invoke_model(
        modelId="anthropic.claude-3-5-sonnet-20240620-v1:0",
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 1024,
            "messages": [{"role": "user", "content": prompt}],
        }),
    )
    return json.loads(resp["body"].read())["content"][0]["text"]
```

`boto3` picks the bearer token up from the environment automatically — you do not need AWS credentials on the machine for this. Requires `boto3 >= 1.39`; run `pip install -U boto3` if `invoke_model` returns an auth error.

One key per team. Do not share it between teams, and do not commit it.

**A note on latency:** Singapore is a long way from São Paulo — expect a few hundred milliseconds per round trip. Fine for a demo, worth knowing if you are chaining many calls in a loop. Batch your calls; do not chain them one at a time inside a loop.

**Nothing here is prescriptive.** Use a different model, a different provider, or no LLM API at all if your idea is better served that way. Explore.

### TiDB vector search — the short way

TiDB can generate the embeddings for you, so you never call an embedding API from your own code — and no Bedrock key is involved:

```sql
CREATE TABLE flight_notes (
  id BIGINT AUTO_RANDOM PRIMARY KEY,
  flight_id INT,
  note TEXT,
  embedding VECTOR(1024) GENERATED ALWAYS AS (
    EMBED_TEXT("tidbcloud_free/amazon/titan-embed-text-v2", note)
  ) STORED
);

SELECT flight_id, note FROM flight_notes
ORDER BY VEC_COSINE_DISTANCE(
  embedding,
  EMBED_TEXT("tidbcloud_free/amazon/titan-embed-text-v2", 'nonstop to Germany')
) LIMIT 5;
```

Insert text, get vectors.

---

## Setup, On Site

Nothing happens before the event. This block runs at the venue, **before the 2h30 build clock starts**.

1. **Get your team.** Teams are formed on the morning from the confirmed list and announced at the venue. The number you are given — `latam-hackathon-0XX` — is your AWS username, your EC2 tag and your S3 prefix, so everything else depends on it.
2. **Sign in to AWS** and enrol MFA. Organizers verify this before handing out keys.
3. **Collect your Bedrock API key** — issued for `ap-southeast-1`.
4. **Install Kiro** from [kiro.dev](https://kiro.dev) and sign in.
5. **Register your team's TiDB Cloud Starter cluster** — free, about a minute.
6. **Import the dataset** into that cluster.

---

## What We Provide, On Site

Nothing below is handed out in advance. Collect it from an organizer at the venue, during the setup block.

| Item | Per | Notes |
|---|---|---|
| **AWS account** | one per team | IAM user `latam-hackathon-0XX`. Temporary password, changed on first sign-in, **MFA required**. |
| **Bedrock API key** | one per team | Valid in **ap-southeast-1**, not sa-east-1. Issued after MFA is confirmed. One key per team — never shared. |
| **EC2 instance** | one per team | sa-east-1, pre-tagged to your team. |
| **S3 folder** | one per team | `s3://tidb-latam-hackathon-2026-048364544505/latam-hackathon-0XX/` |
| **airportdb dataset** | one per team | Import into your own TiDB Cloud Starter cluster. |

You register your own **TiDB Cloud Starter** cluster — it is free and takes about a minute. Organizers never hold your TiDB credentials; they are yours from the moment you create the cluster.

Your AWS username, your EC2 `Participant` tag and your S3 prefix all use the same hyphenated form, `latam-hackathon-0XX`. The EC2 tag is called `Participant` for historical reasons; the value in it is your team's username.

**Key discipline:** keys live in the `.env` file on your machine or EC2, and nowhere else. Never in a commit, a README, a screenshot, or a Dockerfile. All keys are revoked when the event ends.

---

## How to Submit

**Build in your team's own public GitHub repository. Do not fork this repo, and do not open a pull request here.**

1. Work in a repository your team owns, and create it **public**.
2. It must contain your code, your `.kiro/` specs if you used Kiro, and a `SUBMISSION.md` at the root: what you built, how to run it, what you would do next.
3. When you are done, comment on the pinned **Entregas** issue in this repository with three things: your team name, your repository URL, and your demo link — a 2-minute video or a live URL.
4. That is the entire submission.

### The freeze — read this before you comment

At the deadline, organizers **fork every submitted repository**. That fork is the snapshot that gets judged.

- A fork does not sync. Anything you push after the deadline is **not judged**. Push everything first, then comment.
- Check the repository is actually **public** before you comment — a private repo cannot be forked, so it cannot be scored.

---

## Rules

- `main` is protected. Only organizers can merge.
- Secret Scanning and Push Protection are enabled. Never commit an API key, TiDB password, or AWS credential.
- Each team gets one AWS account, one EC2 instance, one S3 prefix, and one model API key, shared by that team. Do not share credentials between teams.
- Enroll MFA on first sign-in. Organizers verify this before handing out the Bedrock API key.
- By submitting, your team agrees that organizers keep an archived copy of the repository — the fork made at the deadline.

---

## Resources

| What | Where |
|---|---|
| AWS Console | `https://tidb-latam-hackathon.signin.aws.amazon.com/console` |
| AWS Region | `sa-east-1` (São Paulo) |
| Shared S3 bucket | `s3://tidb-latam-hackathon-2026-048364544505/<your-username>/` |
| TiDB Cloud | <https://tidbcloud.com> — register your own Starter cluster |
| TiDB docs | <https://docs.pingcap.com/tidbcloud/> |
| Kiro | <https://kiro.dev> — install it before the event |
| Bedrock region | `ap-southeast-1` — Claude 3.5 Sonnet · Claude 3 Haiku · Cohere Embed v3 |
| Event brief | [Ask the Airport.pdf](Ask%20the%20Airport.pdf) |
| Session Manager plugin | <https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html> |

---

## Getting Help

Ask an organizer on site. When reporting a problem, include: your account name, the region, what you ran, and the exact error text.
