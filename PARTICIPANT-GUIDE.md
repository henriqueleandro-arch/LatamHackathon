# Participant Guide — TiDB LATAM Hackathon 2026

> Replace every `latam-hackathon-0XX` below with your team's account name. Ten teams run this event: `latam-hackathon-001` through `latam-hackathon-010`.

Event ends **2026-09-05**. All accounts and resources are deleted after that date.

---

## 1. Sign In

**Your team is assigned at the venue.** Teams are formed on the morning of the event from the confirmed attendee list. The number you are handed — `latam-hackathon-0XX` — identifies your AWS user, your EC2 instance tag and your S3 prefix. Nothing below works until you have it, and nothing here needs doing before the event.

```
https://tidb-latam-hackathon.signin.aws.amazon.com/console
```

- **Username:** `latam-hackathon-0XX` — one account per team, shared by the whole team.
- You will be forced to change your password on first sign-in (minimum 14 characters, with uppercase, lowercase, number and symbol).
- Immediately after changing your password, **enroll an MFA device**: top-right user menu → *Security credentials* → *Assign MFA device*.
- Organizers will not hand out your team's Bedrock API key until MFA is enrolled. TiDB credentials do not come from us — your team registers its own TiDB Cloud Starter cluster (section 6).

Your region is fixed to **sa-east-1 (São Paulo)**. Switching to any other region shows nothing and every call is denied.

> **One deliberate exception:** your Bedrock key is issued for **ap-southeast-1 (Singapore)**. Console, EC2 and S3 are São Paulo; Bedrock calls go to Singapore. This is expected, not a bug — section 4 has the copy-paste config.

---

## 2. EC2

One instance per team. You can only operate the instance tagged `Participant = <your team's username>` — the tag is named `Participant`, but the value it holds is your team's account name.

| You can | You cannot |
|---|---|
| List instances | Launch or terminate instances |
| Start / stop / reboot **your team's** instance | Touch another team's instance |
| Connect to your team's instance via Session Manager | Modify security groups, VPC, routes, EIPs |

### Connecting

There is **no SSH port open** and no key pair. Use **Session Manager**.

**Console:** EC2 → select your instance → *Connect* → *Session Manager* → *Connect*

**Local terminal** (install the [Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) first):

```bash
aws ssm start-session --target <your-instance-id> --region sa-east-1
```

**VS Code Remote SSH** over an SSM tunnel — add to `~/.ssh/config`:

```
Host hackathon
  HostName <your-instance-id>
  User ec2-user
  ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p --region sa-east-1"
```

### Deploying

Code moves in through GitHub, not from your laptop — you have no AWS access keys, and that is deliberate.

```bash
# 1. Tools — neither git nor pip is preinstalled
sudo dnf install -y git python3-pip

# 2. The system Python is 3.9, which boto3 no longer supports. Install 3.11:
sudo dnf install -y python3.11 python3.11-pip

# 3. Your code — clone over HTTPS, port 443 is open
git clone https://github.com/<you>/<your-repo>.git
cd <your-repo>
python3.11 -m pip install -r requirements.txt

# 4. Your keys — on the instance only, never committed
cat > .env <<'ENV'
AWS_BEARER_TOKEN_BEDROCK=<your team's Bedrock key>
AWS_REGION=ap-southeast-1
TIDB_HOST=gateway01.<region>.prod.aws.tidbcloud.com
TIDB_PASSWORD=<your password>
ENV
chmod 600 .env

# 5. Run it so it survives the session closing
setsid nohup python3.11 src/main.py > app.log 2>&1 < /dev/null &
```

`AWS_REGION=ap-southeast-1` is correct and intentional — that is the Bedrock region (Singapore), not the console region.

Use a heredoc for `.env` rather than pasting into `vi` — the browser shell mangles indentation.

Things that will bite you otherwise:

| | |
|---|---|
| **913 MB RAM**, no swap | `t3.micro`. pandas is fine; torch or large in-memory embeddings will be killed by the OOM reaper. Process in batches. |
| **Session closes → process dies** | Use `setsid nohup ... &` as above, or `tmux`. |
| **20-minute idle timeout** | The browser shell disconnects when idle. Your backgrounded process keeps running. |
| **`git push` over SSH fails from here** | Outbound SSH is closed, so a `git@github.com:...` remote will not work. Push over **HTTPS** instead — port 443 is open: `git remote set-url origin https://github.com/<you>/<your-repo>.git`, then use a personal access token when prompted. |
| **Public IP changes on restart** | No Elastic IP. Re-check it in the console after every stop/start. |


---

## 3. S3

Every team shares one bucket. Each team owns one prefix:

```
s3://tidb-latam-hackathon-2026-048364544505/latam-hackathon-0XX/
```

**Always work inside your team's own folder.** Creating a folder or uploading a file at the bucket root is denied — this is intentional, not a bug.

### Console

Open the bucket and you will see every team's folder name, but you can only open your own. Clicking into another team's returns *Access Denied* — expected.

Direct link to your folder:

```
https://sa-east-1.console.aws.amazon.com/s3/buckets/tidb-latam-hackathon-2026-048364544505?region=sa-east-1&prefix=latam-hackathon-0XX/
```

### CLI

```bash
BUCKET=s3://tidb-latam-hackathon-2026-048364544505/latam-hackathon-0XX

# Upload
aws s3 cp ./model.pkl $BUCKET/ --region sa-east-1

# List your folder
aws s3 ls $BUCKET/ --region sa-east-1

# Download
aws s3 cp $BUCKET/model.pkl . --region sa-east-1
```

### Expected denials

| Action | Result |
|---|---|
| Create your own bucket | Denied — `s3:CreateBucket` is not granted |
| Create a folder at the bucket root | Denied |
| Read or write another team's folder | Denied |
| `aws s3 ls s3://<bucket>/ --recursive` | Denied — recursive listing of the whole bucket |

Signing out and back in will **not** change any of this. Ask an organizer if you believe you need more.

---

## 4. Amazon Bedrock

**Your key is issued for `ap-southeast-1` (Singapore), not São Paulo.** The console, your EC2 instance and your S3 folder are `sa-east-1`; Bedrock alone is Singapore. That split is deliberate. Point a client at `sa-east-1` and the key will not authenticate — you get an authentication or access-denied error, not an empty result. If you see that error, check the region before anything else.

Copy-paste config, region already correct — never in a commit, a screenshot or a Dockerfile:

```bash
# .env  —  keep it git-ignored
AWS_BEARER_TOKEN_BEDROCK=<the key handed to your team on site>
AWS_REGION=ap-southeast-1
```

```python
import boto3

# Singapore. Not sa-east-1. The key authenticates nowhere else.
bedrock = boto3.client("bedrock-runtime", region_name="ap-southeast-1")
```

`boto3` reads the bearer token straight from the environment, so you do not need AWS credentials on the machine for this. Requires `boto3 >= 1.39`.

Calling Singapore from São Paulo costs a few hundred milliseconds per call. Batch your requests; do not chain one call per row inside a loop.

Available on-demand models:

| Purpose | Model ID |
|---|---|
| Text | `anthropic.claude-3-5-sonnet-20240620-v1:0` |
| Text (faster) | `anthropic.claude-3-haiku-20240307-v1:0` |
| Embeddings | `cohere.embed-english-v3` · `cohere.embed-multilingual-v3` — 1024 dims |

`amazon.titan-embed-text-v2:0` and the Mistral models do **not** exist in ap-southeast-1. For Titan embeddings use TiDB's built-in `EMBED_TEXT()` instead — it never touches Bedrock, needs no key and no region, and it is the easiest path to vector search.

You are free to use a different model or a different provider entirely. Explore.

---

## 5. The Dataset — `airportdb`

A curated airline database: airports, flights, bookings, passengers and weather, filtered to flights touching Brazil across one week in June 2015. Using it is optional, but it is there because it is **rich in real problems** — delays cascade, connections break, weather disrupts, passengers have histories.

### Download

```bash
curl -O https://hackaton-tidb.s3.sa-east-1.amazonaws.com/dumps/hackathon_airportdb.sql.gz
gunzip hackathon_airportdb.sql.gz
```

10 MB compressed, ~30 MB of SQL. No credentials needed — the link is public.

### Load it into your TiDB cluster

```bash
mysql -h <your-tidb-host> -P 4000 -u <your-user> -p \
      --ssl-ca=/etc/ssl/cert.pem \
      -e "CREATE DATABASE IF NOT EXISTS airportdb"

mysql -h <your-tidb-host> -P 4000 -u <your-user> -p \
      --ssl-ca=/etc/ssl/cert.pem airportdb < hackathon_airportdb.sql
```

TLS is required on the public endpoint — without `--ssl-ca` you get an SSL error.

If the venue wifi makes the import crawl, use **TiDB Cloud's import-from-S3** in the console instead. The cluster pulls the file server-side and never touches your connection.

### What is in it

| Table | Rows | What it holds |
|---|---:|---|
| `booking` | 617,062 | seat, price, passenger, flight |
| `airport` / `airport_geo` | 9,854 each | codes, names, city, country, lat/lon |
| `flightschedule` | 9,881 | the planned timetable, by weekday |
| `weatherdata` | 9,216 | temp, humidity, pressure, wind, condition |
| `flight` | 5,191 | actual departures and arrivals |
| `airplane` | 5,583 | capacity, type, airline |
| `passenger` / `passengerdetails` | 36,095 each | name, passport, birthdate, address, email |
| `employee` | 1,000 | staff records |
| `airline` | 113 | IATA code, name, base airport |
| `airplane_type` | 342 | aircraft identifiers |

Flights run **2015-06-02 to 2015-06-09**. 12 tables in total.

> The event brief mentions 8 tables and ~500K bookings. The dump actually carries **12 tables and 617,062 bookings** — the numbers above are counted from the file itself.

### Where the interesting questions are

Spend a few minutes querying before you write code. The strong projects come from a pattern someone noticed in the data, not from a feature list.

- `flight` versus `flightschedule` — where does reality diverge from the timetable, and for which routes?
- `weatherdata` joined to departure times — does weather actually explain the delays, or is something else going on?
- `booking` per passenger over the week — what does someone's history say about what they would accept as a rebooking?
- Connection windows: which itineraries break when the inbound leg slips an hour?

`passengerdetails` contains names, addresses and email addresses. It is synthetic data, but treat it as if it were not: do not paste it into a public repo, a screenshot or a prompt log.

---

## 6. TiDB Cloud

Each team registers its own **TiDB Cloud Starter** cluster and connects over the public endpoint with TLS. Organizers never hold these credentials — your team creates them.

The default access setting on a new Starter cluster is fine for a 2h30 event on a throwaway cluster. **Do not pin IP addresses.** Your EC2's public IP changes on every stop/start (there is no Elastic IP), so an IP Access List will silently cut your app off from its database mid-sprint.

What does matter: the host, user and password are credentials. Keep them in the `.env` on your instance, and never commit them.

---

## 7. Network Limits

### Inbound — your app is reachable

| Port | For |
|---|---|
| 8000–8999 | Your web app — Streamlit (8501), FastAPI/Django (8000), anything in the range |
| 3000 | React / Next.js dev server |

Open to the whole internet, so you can share a link with judges and teammates. Start your app bound to **`0.0.0.0`**, not `127.0.0.1`, or nothing outside the instance can reach it:

```bash
streamlit run app.py --server.address 0.0.0.0 --server.port 8501
uvicorn main:app --host 0.0.0.0 --port 8000
```

Then open `http://<your-instance-public-ip>:8501` in a browser. Find the IP in the EC2 console under **Public IPv4 address**.

> ⚠️ **The public IP changes every time the instance stops and starts.** Re-check it after any restart.

> ⚠️ **Anything you serve here is public.** No auth, no TLS, and scanners find open ports within hours. Do not put credentials, personal data or anything you would not publish behind these ports.

**SSH (22) and RDP (3389) stay closed** and will not be opened. Use Session Manager.

### Outbound — three ports only

| Port | For |
|---|---|
| 443 | HTTPS — GitHub, pip/npm, Bedrock, TiDB Cloud |
| 80 | HTTP — package repositories |
| 4000 | TiDB Cloud public endpoint |

Everything else is blocked. One consequence worth knowing: **outbound SSH is closed**, so a `git@github.com:...` remote will not work from the instance. Use an **HTTPS remote** — `https://github.com/<you>/<your-repo>.git` over port 443 — and both `git pull` and `git push` work from here, authenticating with a personal access token.

---

## 8. What You Do Not Have

Launching or terminating EC2, modifying security groups or VPC, managing any IAM user or policy, granting yourself permissions, reading Secrets Manager, and touching another team's EC2 or S3 folder.

Need something else? Ask an organizer. Do not attempt to work around the limits — CloudTrail logs everything.

---

## 9. API Key Discipline

- Bedrock keys are handed out **on site**, one per team. Never share your key outside your team.
- Store keys in the `.env` file on your EC2 only.
- Never put a key in GitHub, a README, a screenshot, a Dockerfile, or a container image.
- All keys are revoked when the event ends.

This repository has **Secret Scanning and Push Protection enabled**. A push containing a key will be blocked.

---

## 10. Submitting Your Work

**Build in your team's own public GitHub repository, then tell us where it is.** Do not fork the main repository and do not open a pull request there.

> **Kiro is installed on the day**, during the setup block — get it from [kiro.dev](https://kiro.dev). It is not preinstalled on your instance. Building with Kiro is recommended, and committing your `.kiro/` specs scores points.
>
> **If you ask Kiro to pick the problem, it will pick the same one as the other nine teams. Choosing the problem is yours to make.**

1. Build in your team's own **public** GitHub repository. Move fast, do not wait on reviews.
2. The repo must contain your code, your `.kiro/` specs if you used Kiro, and a `SUBMISSION.md` at the root — what you built, how to run it, what you would do next.
3. When you are done, comment on the pinned **"Entregas"** issue in the main repository with your team name, your repository URL, and your demo link (a 2-minute video or a live URL).
4. Done. That is the whole submission.

### The deadline is a freeze

Organizers fork every submitted repo at the deadline, and **that fork is what gets judged**.

- Forks do not sync. Anything you push after the deadline is not judged. **Push first, then comment.**
- Verify the repo is genuinely public before you comment — a private repo cannot be forked and cannot be scored.
- By submitting, your team agrees that organizers keep an archived copy of the repository (the fork).

Leave `.env` out of the repository, along with anything else secret. Your own repo and this one both have **Secret Scanning and Push Protection enabled**: a push containing a key will be blocked.

---

## 11. Cleanup — 2026-09-05

After the event, all of the following are permanently deleted: your team's AWS account, its EC2 instance and disk, its S3 folder and contents, TiDB clusters, and every API key.

**Push anything you want to keep to your team's GitHub repository before that date.**
