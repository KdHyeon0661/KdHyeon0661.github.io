---
layout: post
title: flask - Flask CLI & 스크립팅
date: 2025-09-21 23:25:23 +0900
category: flask
---
# 부록 A. Flask CLI & 스크립팅

> 이 부록은 **Flask CLI**를 이용해 운영에 필요한 **커스텀 커맨드**를 만들고, **마이그레이션/시드 데이터**를 안전하게 적용하며, **크론 대체(스케줄링)** 를 구현/운영하는 법을 **끝까지** 정리한다.

---

## A.1 왜 Flask CLI인가?

- **일관된 앱 컨텍스트**: `create_app()` 를 통해 ORM/캐시/설정에 접근.
- **운영 표준화**: 마이그레이션, 시드, 백필(Backfill), 리페어(Repair), 배치(ETL) 작업을 **한 엔트리포인트**에서.
- **테스트/배포 용이성**: CI/CD, K8s Job, GitHub Actions, Docker 컨테이너에서 **동일 커맨드** 재사용.

---

## A.2 프로젝트 스캐폴딩(권장 레이아웃)

```
app/
  __init__.py          # create_app(config_name)
  extensions.py        # db, migrate, cache, etc.
  models/              # SQLAlchemy models
  services/            # domain/application services
  cli/
    __init__.py        # register_cli(app)
    db.py              # db commands: migrate/seed/backup
    ops.py             # ops commands: backfill/repair
    cron.py            # on-demand scheduled jobs
  scripts/             # pure python scripts reused by CLI
migrations/            # Alembic (Flask-Migrate)
tests/
  test_cli_*.py
```

`app/__init__.py`:

```python
from flask import Flask
from .extensions import db, migrate, cache
from .cli import register_cli

def create_app(config_name="production"):
    app = Flask(__name__)
    # ... app.config.from_object(...)

    db.init_app(app)
    migrate.init_app(app, db)
    cache.init_app(app)

    # 라우트/블루프린트 등록 ...
    register_cli(app)
    return app
```

---

## A.3 Flask CLI 빠른 시작

### A.3.1 실행 방법

```bash
# 환경변수 이용

export FLASK_APP="app:create_app('production')"
flask --help

# --app 인자

flask --app "app:create_app('production')" --debug shell

# Docker 컨테이너 내부

docker exec -it myapp-web flask --app app:create_app
```

> 팁: 개발/스테이징/프로덕션 별로 `FLASK_ENV`, `APP_ENV` 또는 `--app "app:create_app('staging')"` 를 구분해준다.

### A.3.2 Shell 컨텍스트

```python
# app/cli/__init__.py

from flask import current_app
from click import group
from .db import db_cli
from .ops import ops_cli
from .cron import cron_cli

def register_cli(app):
    # shell context: flask shell 에서 미리 import된 이름
    @app.shell_context_processor
    def make_shell_ctx():
        from app.extensions import db
        from app.models import user as user_models
        return {"db": db, **user_models.__dict__}

    # 그룹 등록
    app.cli.add_command(db_cli)
    app.cli.add_command(ops_cli)
    app.cli.add_command(cron_cli)
```

---

## A.4 커스텀 커맨드 만들기 (Click 기반)

### A.4.1 기본 형태

```python
# app/cli/ops.py

import click
from flask import current_app
from app.extensions import db
from app.models.user import User

@click.group("ops")
def ops_cli():
    """운영/유틸리티 명령 모음"""
    pass

@ops_cli.command("hello")
@click.option("--name", default="world", help="인사 대상")
def hello(name):
    click.echo(f"Hello, {name}!")
```

실행:

```bash
flask ops hello --name Alice
```

### A.4.2 앱 컨텍스트에서 DB 사용

```python
@ops_cli.command("deactivate-inactive-users")
@click.option("--days", type=int, default=365, show_default=True)
@click.option("--dry-run/--apply", default=True, help="실제 반영 여부")
def deactivate_inactive_users(days, dry_run):
    """N일 동안 로그인 없는 계정 비활성화"""
    from datetime import datetime, timedelta, timezone
    cutoff = datetime.now(timezone.utc) - timedelta(days=days)
    q = User.query.filter(User.last_login_at < cutoff, User.active.is_(True))

    click.echo(f"Target users: {q.count()}")
    if dry_run:
        for u in q.limit(20):
            click.echo(f"[DRY] deactivate {u.id} {u.email}")
        return

    updated = 0
    for u in q.yield_per(1000):
        u.active = False
        updated += 1
        if updated % 1000 == 0:
            db.session.commit()
    db.session.commit()
    click.echo(f"Deactivated {updated} users")
```

> 실무 포인트
> - **`--dry-run` 기본**으로 안전장치.
> - 대량 업데이트는 **청크 커밋**(`yield_per`)로 트랜잭션·락 부담 완화.
> - 출력은 **표준 출력** → 로그 수집기에서 집계 가능.

### A.4.3 프로그레스 바/확인 프롬프트

```python
@ops_cli.command("recompute-scores")
@click.confirmation_option(prompt="대량 재계산을 실행할까요?")
def recompute_scores():
    from app.models.score import Score
    total = Score.query.count()
    with click.progressbar(Score.query.yield_per(1000), length=total, label="Recomputing") as bar:
        for s in bar:
            s.value = compute_new_value(s)
            if bar.pos % 1000 == 0:
                db.session.commit()
    db.session.commit()
    click.secho("done", fg="green", bold=True)
```

---

## A.5 마이그레이션(Alembic/Flask-Migrate)

### A.5.1 초기화와 사용

```bash
# 설치

pip install Flask-Migrate

# 초기 생성(이미 존재한다면 생략)

flask db init

# 모델 변경 → 자동 마이그레이션 스크립트

flask db migrate -m "add user.active index"

# DB에 반영

flask db upgrade

# 되돌리기

flask db downgrade -1
```

`app/extensions.py`:

```python
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

db = SQLAlchemy(session_options={"autoflush": False, "expire_on_commit": False})
migrate = Migrate()
```

`app/__init__.py` 에서 `migrate.init_app(app, db)` 이미 호출.

### A.5.2 안전 가이드

- **Additive-first**: 컬럼/인덱스 추가 → 코드 배포 → **파괴적 변경은 후속 릴리스**(16장 참조).
- 롤링 배포 중엔 **옛/새 코드 모두 동작** 가능한 스키마를 유지.
- 대용량 마이그레이션은 **오프피크** + **청크/병렬** + **트랜잭션 분리**.

예: 큰 테이블에 인덱스 추가(전용 스크립트로 온라인 수행):

```python
# migrations/versions/xxxx_add_index_concurrently.py

from alembic import op

def upgrade():
    op.execute('CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_last_login ON users (last_login_at)')

def downgrade():
    op.execute('DROP INDEX CONCURRENTLY IF EXISTS idx_users_last_login')
```

> **PostgreSQL**의 `CONCURRENTLY` 는 트랜잭션 블록 밖에서만 가능 → `alembic.ini` 에서 `transaction_per_migration = True` 확인, 필요 시 `op.get_bind().execution_options(isolation_level="AUTOCOMMIT")` 사용.

---

## A.6 시드(Seed) 데이터 — 안전하고 멱등하게

### A.6.1 원칙

- **멱등성**: 여러 번 실행해도 결과가 같아야 한다.
- **환경 별 분기**: dev/staging/prod 시드가 다를 수 있다.
- **변경 가능성**: 시드도 **버전 관리**(마이그레이션과 연동).

### A.6.2 시드 커맨드(기본)

```python
# app/cli/db.py

import click
from flask import current_app
from app.extensions import db
from app.models.user import User, Role

@click.group("dbx")
def db_cli():
    """DB 확장 유틸 (seed/backup/restore 등)"""
    pass

@db_cli.command("seed")
@click.option("--env", type=click.Choice(["dev","staging","prod"]), default="dev")
@click.option("--force/--no-force", default=False, help="존재시 업데이트")
def seed(env, force):
    click.echo(f"Seeding for {env} ...")
    ensure_role("admin")
    ensure_role("member")
    ensure_admin("admin@example.com", force=force)
    db.session.commit()
    click.secho("Seed done", fg="green")

def ensure_role(name: str):
    r = Role.query.filter_by(name=name).one_or_none()
    if not r:
        db.session.add(Role(name=name))

def ensure_admin(email: str, force: bool):
    u = User.query.filter_by(email=email).one_or_none()
    if not u:
        u = User(email=email, name="Admin", active=True)
        db.session.add(u)
    if force:
        u.active = True
        u.name = "Admin"
```

### A.6.3 대량 시드(파일/CSV/JSON)

```python
@db_cli.command("seed-users")
@click.option("--file", type=click.Path(exists=True), required=True)
@click.option("--batch", type=int, default=1000)
def seed_users(file, batch):
    import csv
    from app.models.user import User
    with open(file, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        buf, n = [], 0
        for row in reader:
            buf.append(User(email=row["email"], name=row["name"], active=True))
            if len(buf) >= batch:
                db.session.bulk_save_objects(buf)
                db.session.commit()
                n += len(buf); buf.clear()
        if buf:
            db.session.bulk_save_objects(buf)
            db.session.commit()
            n += len(buf)
    click.echo(f"Seeded {n} users")
```

> **주의**
> - `bulk_save_objects` 는 이벤트 훅/하이브리드 속성을 건너뛸 수 있다. 필요한 경우 일반 `session.add_all` 사용.
> - 중복 키 충돌 방지: **UPSERT**(PG `ON CONFLICT`) 를 Raw SQL 로 사용하거나, **존재 체크** + 업데이트.

---

## A.7 크론 대체(스케줄링) — 전략 비교

| 전략 | 설명 | 장점 | 단점/유의 |
|---|---|---|---|
| **Kubernetes CronJob** | 컨테이너로 일정 실행 | 인프라 표준, 관측/재시도 | K8s 의존 |
| **Celery Beat** | Celery 스케줄러 + 작업 큐 | 작업 재시도/분산 | 워커/브로커 필요 |
| **APScheduler** | 앱 내 임베디드 스케줄러 | 설정 간단, 코드 근접 | 프로세스 수 만큼 중복 실행 방지 필요 |
| **systemd timer** | VM/호스트의 타이머 | 간단/안정 | 컨테이너/클러스터/다중 인스턴스 시 복잡 |
| **GitHub Actions schedule** | 리포지토리 크론 | 운영 외부화, 간단 | 사설 네트워크 접근/비밀 관리 고려 |

### A.7.1 Kubernetes CronJob 예

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-maintenance
spec:
  schedule: "0 3 * * *"          # 매일 03:00
  concurrencyPolicy: Forbid      # 중복 실행 금지
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: cli
              image: myrepo/myapp:web-2025-10-20
              envFrom:
                - secretRef: {name: app-secret}
                - configMapRef: {name: app-config}
              command: ["flask","ops","recompute-scores","--yes"]
```

> **권장**: 운영은 가능하면 **CronJob**. 재시도/관측/격리가 표준화된다.

### A.7.2 Celery Beat + Worker

```python
# app/celery_app.py

from celery import Celery
celery = Celery(__name__, broker="redis://cache:6379/0")
celery.conf.beat_schedule = {
  "sync-stats": {
    "task": "app.tasks.sync_stats",
    "schedule": 300.0,  # 5분마다
  }
}
```

```python
# app/tasks.py

from .celery_app import celery
@celery.task(bind=True, autoretry_for=(Exception,), retry_backoff=2, max_retries=5)
def sync_stats(self):
    # API 호출/DB 업데이트
    ...
```

실행:

```bash
celery -A app.celery_app.celery worker -l INFO
celery -A app.celery_app.celery beat -l INFO
```

### A.7.3 APScheduler (단일 인스턴스에서만)

```python
# app/cli/cron.py

import click
from apscheduler.schedulers.blocking import BlockingScheduler
from flask import current_app
from app.extensions import db

@click.group("cron")
def cron_cli():
    """앱 내장 스케줄러(개발/소규모용)"""
    pass

@cron_cli.command("run")
@click.option("--job", type=click.Choice(["cleanup","sync"]), multiple=True)
def run(job):
    sched = BlockingScheduler(timezone="Asia/Seoul")

    @sched.scheduled_job("cron", minute="*/15", id="cleanup")
    def cleanup():
        current_app.logger.info("running cleanup")
        # ... DB 정리

    @sched.scheduled_job("cron", hour="3", id="sync")
    def sync():
        # ... 외부와 동기화

    # 특정 job만 선택적으로 실행
    if job:
        all_jobs = {"cleanup","sync"}
        for j in all_jobs - set(job):
            sched.remove_job(j)
    sched.start()
```

> **주의**: **복수 인스턴스에서 동시에 돌면 중복 실행**. 리더 선출/분산락(예: Redis Redlock) 없이 운영에 쓰지 말 것.

### A.7.4 systemd timer (VM/베어메탈)

```
# /etc/systemd/system/myapp-cron.service

[Unit]
Description=MyApp CLI cron

[Service]
Environment=FLASK_APP=app:create_app('production')
WorkingDirectory=/srv/myapp
ExecStart=/usr/bin/flask ops recompute-scores --yes
User=myapp
Group=myapp
```

```
# /etc/systemd/system/myapp-cron.timer

[Unit]
Description=MyApp CLI cron timer

[Timer]
OnCalendar=*:0/15
Persistent=true

[Install]
WantedBy=timers.target
```

---

## A.8 운영 친화적인 CLI 패턴

### A.8.1 공통 옵션/로깅/요청-ID

```python
# app/cli/common.py

import click, logging, sys, uuid
@click.group("ops")
@click.option("--verbose", is_flag=True, help="자세한 로그")
@click.pass_context
def ops_cli(ctx, verbose):
    level = logging.DEBUG if verbose else logging.INFO
    logging.basicConfig(stream=sys.stdout, level=level, format="%(asctime)s %(levelname)s %(message)s")
    ctx.obj = {"request_id": uuid.uuid4().hex}
```

하위 커맨드에서:

```python
@ops_cli.command("rebuild-search-index")
@click.option("--index", default="items")
@click.pass_context
def rebuild_index(ctx, index):
    rid = ctx.obj["request_id"]
    logging.info(f"[{rid}] rebuild index={index}")
    # ...
```

### A.8.2 트랜잭션 경계 & 롤백 스위치

```python
@db_cli.command("dangerous-operation")
@click.option("--apply", is_flag=True, help="실제 반영")
def dangerous_operation(apply):
    try:
        with db.session.begin():
            # ... 다량 수정
            if not apply:
                raise RuntimeError("dry-run rollback")
    except RuntimeError as e:
        click.echo(f"Rolled back: {e}")
```

### A.8.3 재시도/백오프(외부 API)

```python
from tenacity import retry, wait_exponential_jitter, stop_after_attempt

@retry(wait=wait_exponential_jitter(0.2, 2.0), stop=stop_after_attempt(5), reraise=True)
def fetch_chunk(page: int):
    # httpx.get(...).raise_for_status().json()
    ...
```

---

## A.9 데이터 백필/정합성 리페어(실전)

### A.9.1 안전한 백필 방식

- **배치 범위**: PK 범위/시간 범위로 분할 (`id BETWEEN a AND b`)
- **체크포인트**: 진행 상태를 **DB/파일**에 기록(중단/재개 가능)
- **리드 레플리카** 읽기 + **마스터 쓰기** (필요 시)
- **Idempotent**: 같은 레코드 재처리해도 안전하도록 `UPSERT`/버전 체크

예:

```python
@ops_cli.command("backfill-user-slugs")
@click.option("--start-id", type=int, default=1)
@click.option("--end-id", type=int, required=True)
@click.option("--chunk", type=int, default=10000)
def backfill_user_slugs(start_id, end_id, chunk):
    from app.models.user import User
    from sqlalchemy import and_
    for lo in range(start_id, end_id+1, chunk):
        hi = min(lo + chunk - 1, end_id)
        q = User.query.filter(and_(User.id>=lo, User.id<=hi), User.slug.is_(None))
        with click.progressbar(q.yield_per(1000), label=f"{lo}-{hi}") as bar:
            for u in bar:
                u.slug = slugify(u.name)
            db.session.commit()
```

### A.9.2 정합성 점검/리포트

```python
@ops_cli.command("consistency-report")
def consistency_report():
    from sqlalchemy import text
    sql = text("""
      SELECT COUNT(*) AS orphan_orders
      FROM orders o LEFT JOIN users u ON o.user_id = u.id
      WHERE u.id IS NULL
    """)
    row = db.session.execute(sql).mappings().first()
    click.echo(f"orphan_orders={row['orphan_orders']}")
```

---

## A.10 백업/복구(간단한 자동화 예)

> 실제 백업은 관리형 DB의 스냅샷/Point-In-Time-Recovery(PITR) 를 권장. 아래는 **추가 덤프** 예.

```python
@db_cli.command("backup")
@click.option("--out", type=click.Path(), default="backup.sql.gz")
def backup(out):
    import subprocess
    dsn = current_app.config["DATABASE_URL"]
    cmd = ["pg_dump", dsn, "-Z", "9", "-f", out]
    subprocess.check_call(cmd)
    click.echo(f"backup -> {out}")
```

복구는 사람이 **두 번 확인**하도록:

```python
@db_cli.command("restore")
@click.option("--file", type=click.Path(exists=True), required=True)
@click.confirmation_option(prompt="해당 데이터베이스를 덮어씁니다. 계속하시겠습니까?")
def restore(file):
    import subprocess
    dsn = current_app.config["DATABASE_URL"]
    cmd = ["psql", dsn, "-f", file]
    subprocess.check_call(cmd)
```

---

## A.11 CLI와 테스트(자동화)

### A.11.1 pytest에서 CLI 실행

```python
# tests/test_cli_ops.py

def test_hello(runner):
    result = runner.invoke(args=["ops","hello","--name","Bob"])
    assert result.exit_code == 0
    assert "Hello, Bob!" in result.output
```

`conftest.py` 의 `runner` 픽스처(15장 참고):

```python
@pytest.fixture()
def runner(app):
    return app.test_cli_runner()
```

### A.11.2 시드/백필 테스트

- 작은 **테스트 전용 DB** 에 시드를 넣고, CLI로 백필 실행 → 결과 검사.
- **idempotent** 여부 확인: 같은 커맨드 2회 실행 → 결과 동일.

```python
def test_seed_idempotent(runner, db):
    r1 = runner.invoke(args=["dbx","seed","--env","dev"])
    assert r1.exit_code == 0
    r2 = runner.invoke(args=["dbx","seed","--env","dev"])
    assert r2.exit_code == 0
```

---

## A.12 운영 파이프라인에 녹이기

### A.12.1 GitHub Actions로 주기 작업

{% raw %}
```yaml
# .github/workflows/nightly.yml

name: nightly
on:
  schedule:
    - cron: "0 18 * * *"   # KST 03:00
jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt
      - name: Run maintenance
        env:
          FLASK_APP: "app:create_app('production')"
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: flask ops recompute-scores --yes
```
{% endraw %}

### A.12.2 배포 훅에 마이그레이션/시드

- **K8s 배포 전**: `flask db upgrade` (Job)
- **K8s 배포 후**: 필요 시 `flask dbx seed --env=prod --no-force` (idempotent)

---

## A.13 보안/신뢰성 모범사례(CLI 관점)

- **권한 최소화**: 운영 CLI는 **읽기/쓰기 범위 제한**된 DB 계정 사용.
- **비밀 주입**: 환경변수/Secret만, 코드/레포에 저장 금지.
- **장시간 작업**: **타임아웃/체크포인트/재개** 지원.
- **로깅**: JSON 구조화 + `request_id` + 측정치(처리 건수/초당 처리량/오류 수).
- **알림**: 중요한 결과/오류는 Slack/Webhook 연동.
- **드라이런**: 파괴적 커맨드는 **반드시 `--dry-run` 기본** + `--apply` 필요.

예: Slack 알림(간단)

```python
def notify(text):
    import httpx, os
    url = os.getenv("SLACK_WEBHOOK_URL")
    if url:
        httpx.post(url, json={"text": text}, timeout=5)
```

---

## A.14 흔한 안티패턴

- **create_app 미사용**: Flask 컨텍스트 밖에서 ORM 사용 → 설정/세션 누락.
- **비멱등 시드**: 매 실행마다 중복 생성/충돌.
- **대량 작업 단일 트랜잭션**: 롤백 시 지옥, 락 장시간 유지.
- **APScheduler를 다중 인스턴스에서 무심코 실행**: 중복 수행/경합.
- **Stdout 아닌 파일 로그**: 컨테이너 환경에서 수집 불가.
- **예외 잡지 않음/종료 코드 무시**: CI/K8s에서 실패 감지 불가.

---

## A.15 붙여넣기 스타터 팩

### A.15.1 `app/cli/db.py` (요약)

```python
import click
from app.extensions import db
from app.models import *

@click.group("dbx")
def db_cli(): ...

@db_cli.command("seed")
@click.option("--env", type=click.Choice(["dev","staging","prod"]), default="dev")
@click.option("--force/--no-force", default=False)
def seed(env, force):
    ensure_defaults(force)
    db.session.commit()
    click.echo("ok")
```

### A.15.2 `app/cli/ops.py` (요약)

```python
import click
from app.extensions import db

@click.group("ops")
def ops_cli(): ...

@ops_cli.command("recompute")
@click.option("--apply", is_flag=True)
def recompute(apply):
    # ... compute
    if apply: db.session.commit()
    else: db.session.rollback()
```

### A.15.3 `app/cli/cron.py` (요약)

```python
import click
from apscheduler.schedulers.blocking import BlockingScheduler

@click.group("cron")
def cron_cli(): ...

@cron_cli.command("run")
def run():
    s = BlockingScheduler()
    @s.scheduled_job("interval", minutes=15)
    def job(): ...
    s.start()
```

---

## A.16 마무리

이 부록에서는 **Flask CLI를 실전 운영 도구**로 다루는 방법을 정리했다.

- **Click 기반 커스텀 커맨드**로 운영/유지보수 절차를 코드화하고,
- **Flask-Migrate**를 통해 **안전한 스키마 변경**과 **멱등 시드**를 수행하며,
- 스케줄링은 **Kubernetes CronJob(권장)**, **Celery Beat**, **APScheduler** 등으로 **크론 대체**를 구현했다.
