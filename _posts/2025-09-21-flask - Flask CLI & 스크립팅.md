---
layout: post
title: flask - Flask CLI & 스크립팅
date: 2025-09-21 23:25:23 +0900
category: flask
---
# Flask CLI & 스크립팅

이 부록은 Flask CLI를 활용하여 운영에 필요한 커스텀 명령어를 만들고, 데이터베이스 마이그레이션과 시드 데이터를 안전하게 적용하며, 스케줄링 작업을 구현하고 운영하는 방법을 종합적으로 정리합니다.

## 왜 Flask CLI를 사용해야 하나?

Flask CLI는 애플리케이션 운영을 위한 필수 도구로 다음과 같은 장점을 제공합니다:

- **일관된 애플리케이션 컨텍스트**: `create_app()` 팩토리 함수를 통해 데이터베이스, 캐시, 설정 등 모든 확장 기능에 일관되게 접근할 수 있습니다.
- **운영 작업 표준화**: 마이그레이션, 데이터 시딩, 백필(Backfill), 복구(Repair), 배치(ETL) 작업을 단일 진입점에서 관리할 수 있습니다.
- **테스트와 배포 용이성**: CI/CD 파이프라인, Kubernetes Job, GitHub Actions, Docker 컨테이너 등 다양한 환경에서 동일한 명령어를 재사용할 수 있습니다.

## 프로젝트 구조(권장 레이아웃)

```
app/
  __init__.py          # create_app(config_name)
  extensions.py        # db, migrate, cache 등 확장 기능
  models/              # SQLAlchemy 모델
  services/            # 도메인/애플리케이션 서비스
  cli/
    __init__.py        # register_cli(app)
    db.py              # 데이터베이스 명령: migrate/seed/backup
    ops.py             # 운영 명령: backfill/repair
    cron.py            # 주문형 예약 작업
  scripts/             # CLI에서 재사용되는 순수 Python 스크립트
migrations/            # Alembic (Flask-Migrate)
tests/
  test_cli_*.py
```

**app/__init__.py**:

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

## Flask CLI 빠른 시작

### 실행 방법

```bash
# 환경변수 활용
export FLASK_APP="app:create_app('production')"
flask --help

# --app 인자 활용
flask --app "app:create_app('production')" --debug shell

# Docker 컨테이너 내부 실행
docker exec -it myapp-web flask --app app:create_app
```

> 팁: 개발, 스테이징, 프로덕션 환경별로 `FLASK_ENV`, `APP_ENV` 또는 `--app "app:create_app('staging')"`를 구분하여 사용하세요.

### Shell 컨텍스트 설정

```python
# app/cli/__init__.py

from flask import current_app
from click import group
from .db import db_cli
from .ops import ops_cli
from .cron import cron_cli

def register_cli(app):
    # shell context: flask shell에서 미리 import될 이름 설정
    @app.shell_context_processor
    def make_shell_ctx():
        from app.extensions import db
        from app.models import user as user_models
        return {"db": db, **user_models.__dict__}

    # CLI 그룹 등록
    app.cli.add_command(db_cli)
    app.cli.add_command(ops_cli)
    app.cli.add_command(cron_cli)
```

## 커스텀 커맨드 만들기 (Click 기반)

### 기본 구조

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
@click.option("--name", default="world", help="인사할 대상")
def hello(name):
    click.echo(f"Hello, {name}!")
```

**실행 예시**

```bash
flask ops hello --name Alice
```

### 애플리케이션 컨텍스트에서 데이터베이스 사용

```python
@ops_cli.command("deactivate-inactive-users")
@click.option("--days", type=int, default=365, show_default=True)
@click.option("--dry-run/--apply", default=True, help="실제 반영 여부")
def deactivate_inactive_users(days, dry_run):
    """N일 동안 로그인 없는 계정 비활성화"""
    from datetime import datetime, timedelta, timezone
    cutoff = datetime.now(timezone.utc) - timedelta(days=days)
    q = User.query.filter(User.last_login_at < cutoff, User.active.is_(True))

    click.echo(f"대상 사용자 수: {q.count()}")
    if dry_run:
        for u in q.limit(20):
            click.echo(f"[DRY] 비활성화 대상: {u.id} {u.email}")
        return

    updated = 0
    for u in q.yield_per(1000):
        u.active = False
        updated += 1
        if updated % 1000 == 0:
            db.session.commit()
    db.session.commit()
    click.echo(f"{updated}명의 사용자를 비활성화했습니다")
```

> **실무 포인트**
> - `--dry-run` 기본값 설정으로 안전장치 마련
> - 대량 업데이트는 청크 단위 커밋(`yield_per`)으로 트랜잭션과 락 부담 완화
> - 출력은 표준 출력 사용 → 로그 수집기에서 집계 가능

### 프로그레스 바와 확인 프롬프트

```python
@ops_cli.command("recompute-scores")
@click.confirmation_option(prompt="대량 재계산을 실행할까요?")
def recompute_scores():
    from app.models.score import Score
    total = Score.query.count()
    with click.progressbar(Score.query.yield_per(1000), length=total, label="재계산 중") as bar:
        for s in bar:
            s.value = compute_new_value(s)
            if bar.pos % 1000 == 0:
                db.session.commit()
    db.session.commit()
    click.secho("완료", fg="green", bold=True)
```

## 마이그레이션(Alembic/Flask-Migrate)

### 초기화와 기본 사용법

```bash
# 설치
pip install Flask-Migrate

# 초기화(최초 1회만)
flask db init

# 모델 변경 감지 → 자동 마이그레이션 스크립트 생성
flask db migrate -m "사용자 활성화 인덱스 추가"

# 데이터베이스에 변경사항 적용
flask db upgrade

# 변경사항 롤백
flask db downgrade -1
```

**app/extensions.py**:

```python
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

db = SQLAlchemy(session_options={"autoflush": False, "expire_on_commit": False})
migrate = Migrate()
```

### 안전한 마이그레이션 가이드라인

- **추가 우선(Additive-first)**: 컬럼/인덱스 추가 → 코드 배포 → 파괴적 변경은 후속 릴리스에서 진행
- 롤링 배포 중에는 기존 코드와 새로운 코드 모두 동작 가능한 스키마 유지
- 대용량 마이그레이션은 트래픽이 적은 시간대에 청크 단위/병렬 처리와 트랜잭션 분리로 진행

**대용량 테이블 인덱스 추가 예시**(전용 스크립트로 온라인 수행):

```python
# migrations/versions/xxxx_add_index_concurrently.py

from alembic import op

def upgrade():
    op.execute('CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_last_login ON users (last_login_at)')

def downgrade():
    op.execute('DROP INDEX CONCURRENTLY IF EXISTS idx_users_last_login')
```

> PostgreSQL의 `CONCURRENTLY` 옵션은 트랜잭션 블록 밖에서만 가능합니다. `alembic.ini`에서 `transaction_per_migration = True` 확인이 필요하며, 필요시 `op.get_bind().execution_options(isolation_level="AUTOCOMMIT")`를 사용하세요.

## 데이터 시딩: 안전하고 멱등하게

### 기본 원칙

- **멱등성**: 여러 번 실행해도 동일한 결과 보장
- **환경별 분기**: 개발/스테이징/프로덕션 환경에 따라 다른 시드 데이터 적용 가능
- **변경 관리**: 시드 데이터도 버전 관리 대상(마이그레이션과 연동)

### 기본 시드 커맨드

```python
# app/cli/db.py

import click
from flask import current_app
from app.extensions import db
from app.models.user import User, Role

@click.group("dbx")
def db_cli():
    """데이터베이스 확장 유틸리티 (seed/backup/restore 등)"""
    pass

@db_cli.command("seed")
@click.option("--env", type=click.Choice(["dev","staging","prod"]), default="dev")
@click.option("--force/--no-force", default=False, help="이미 존재하면 업데이트")
def seed(env, force):
    click.echo(f"{env} 환경에 대한 데이터 시딩 시작...")
    ensure_role("admin")
    ensure_role("member")
    ensure_admin("admin@example.com", force=force)
    db.session.commit()
    click.secho("데이터 시딩 완료", fg="green")

def ensure_role(name: str):
    r = Role.query.filter_by(name=name).one_or_none()
    if not r:
        db.session.add(Role(name=name))

def ensure_admin(email: str, force: bool):
    u = User.query.filter_by(email=email).one_or_none()
    if not u:
        u = User(email=email, name="관리자", active=True)
        db.session.add(u)
    if force:
        u.active = True
        u.name = "관리자"
```

### 대량 데이터 시딩(파일/CSV/JSON)

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
    click.echo(f"{n}명의 사용자 데이터 시딩 완료")
```

> **주의사항**
> - `bulk_save_objects`는 이벤트 훅과 하이브리드 속성을 무시할 수 있습니다. 필요한 경우 일반 `session.add_all` 사용
> - 중복 키 충돌 방지를 위해 UPSERT(PostgreSQL `ON CONFLICT`)를 Raw SQL로 사용하거나, 존재 여부 체크 후 업데이트

## 스케줄링 전략 비교

| 전략 | 설명 | 장점 | 단점/고려사항 |
|---|---|---|---|
| **Kubernetes CronJob** | 컨테이너 기반 정기 작업 실행 | 인프라 표준화, 관측성/재시도 내장 | Kubernetes 의존성 |
| **Celery Beat** | Celery 스케줄러 + 작업 큐 | 작업 재시도/분산 처리 지원 | 워커/브로커 인프라 필요 |
| **APScheduler** | 애플리케이션 내장 스케줄러 | 설정 간단, 코드 근접성 | 다중 인스턴스 중복 실행 방지 필요 |
| **systemd timer** | VM/호스트 레벨 타이머 | 간단/안정적 | 컨테이너/클러스터 환경에서 복잡 |
| **GitHub Actions schedule** | 리포지토리 크론 | 운영 외부화, 설정 간단 | 사설 네트워크 접근/비밀 관리 고려 |

### Kubernetes CronJob 예시

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

> 권장: 운영 환경에서는 가능하면 Kubernetes CronJob을 사용하세요. 재시도, 관측성, 격리가 표준화되어 있습니다.

### Celery Beat + Worker

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
    # API 호출/데이터베이스 업데이트 로직
    ...
```

**실행 방법**

```bash
celery -A app.celery_app.celery worker -l INFO
celery -A app.celery_app.celery beat -l INFO
```

### APScheduler (단일 인스턴스 환경용)

```python
# app/cli/cron.py

import click
from apscheduler.schedulers.blocking import BlockingScheduler
from flask import current_app
from app.extensions import db

@click.group("cron")
def cron_cli():
    """애플리케이션 내장 스케줄러(개발/소규모 운영용)"""
    pass

@cron_cli.command("run")
@click.option("--job", type=click.Choice(["cleanup","sync"]), multiple=True)
def run(job):
    sched = BlockingScheduler(timezone="Asia/Seoul")

    @sched.scheduled_job("cron", minute="*/15", id="cleanup")
    def cleanup():
        current_app.logger.info("정리 작업 실행 중")
        # ... 데이터베이스 정리 로직

    @sched.scheduled_job("cron", hour="3", id="sync")
    def sync():
        # ... 외부 시스템과 동기화 로직

    # 특정 작업만 선택적으로 실행
    if job:
        all_jobs = {"cleanup","sync"}
        for j in all_jobs - set(job):
            sched.remove_job(j)
    sched.start()
```

> 주의: 다중 인스턴스 환경에서 동시 실행 시 중복 작업 문제 발생. 리더 선출 또는 분산 락(예: Redis Redlock) 없이 운영 환경에 사용하지 마세요.

### systemd timer (VM/베어메탈 환경)

**서비스 파일** (`/etc/systemd/system/myapp-cron.service`):

```
[Unit]
Description=MyApp CLI 정기 작업

[Service]
Environment=FLASK_APP=app:create_app('production')
WorkingDirectory=/srv/myapp
ExecStart=/usr/bin/flask ops recompute-scores --yes
User=myapp
Group=myapp
```

**타이머 파일** (`/etc/systemd/system/myapp-cron.timer`):

```
[Unit]
Description=MyApp CLI 정기 작업 타이머

[Timer]
OnCalendar=*:0/15
Persistent=true

[Install]
WantedBy=timers.target
```

## 운영 친화적인 CLI 패턴

### 공통 옵션, 로깅, 요청 ID

```python
# app/cli/common.py

import click, logging, sys, uuid
@click.group("ops")
@click.option("--verbose", is_flag=True, help="자세한 로그 출력")
@click.pass_context
def ops_cli(ctx, verbose):
    level = logging.DEBUG if verbose else logging.INFO
    logging.basicConfig(stream=sys.stdout, level=level, format="%(asctime)s %(levelname)s %(message)s")
    ctx.obj = {"request_id": uuid.uuid4().hex}
```

**하위 커맨드에서 사용**

```python
@ops_cli.command("rebuild-search-index")
@click.option("--index", default="items")
@click.pass_context
def rebuild_index(ctx, index):
    rid = ctx.obj["request_id"]
    logging.info(f"[{rid}] 인덱스 재구성 시작: {index}")
    # ... 재구성 로직
```

### 트랜잭션 경계와 롤백 스위치

```python
@db_cli.command("dangerous-operation")
@click.option("--apply", is_flag=True, help="실제 반영 여부")
def dangerous_operation(apply):
    try:
        with db.session.begin():
            # ... 다량 수정 작업
            if not apply:
                raise RuntimeError("드라이런 롤백")
    except RuntimeError as e:
        click.echo(f"롤백 완료: {e}")
```

### 재시도와 백오프(외부 API 호출)

```python
from tenacity import retry, wait_exponential_jitter, stop_after_attempt

@retry(wait=wait_exponential_jitter(0.2, 2.0), stop=stop_after_attempt(5), reraise=True)
def fetch_chunk(page: int):
    # httpx.get(...).raise_for_status().json()
    ...
```

## 데이터 백필과 정합성 복구(실전 사례)

### 안전한 백필 방식

- **배치 범위 지정**: PK 범위/시간 범위로 작업 분할 (`id BETWEEN a AND b`)
- **체크포인트 기록**: 진행 상태를 데이터베이스나 파일에 기록(중단/재개 가능)
- **읽기/쓰기 분리**: 리드 레플리카에서 읽기 + 마스터에 쓰기 (필요시)
- **멱등성 보장**: 동일 레코드 재처리 시에도 안전하도록 UPSERT/버전 체크 구현

**예시**

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

### 정합성 점검과 리포트 생성

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
    click.echo(f"고아 주문 수: {row['orphan_orders']}")
```

## 백업과 복구(간단한 자동화 예시)

> 실제 운영 환경에서는 관리형 데이터베이스의 스냅샷/Point-In-Time-Recovery(PITR) 기능을 권장합니다. 아래는 추가 백업 예시입니다.

```python
@db_cli.command("backup")
@click.option("--out", type=click.Path(), default="backup.sql.gz")
def backup(out):
    import subprocess
    dsn = current_app.config["DATABASE_URL"]
    cmd = ["pg_dump", dsn, "-Z", "9", "-f", out]
    subprocess.check_call(cmd)
    click.echo(f"백업 완료: {out}")
```

**복구 작업**(사용자 확인 강제):

```python
@db_cli.command("restore")
@click.option("--file", type=click.Path(exists=True), required=True)
@click.confirmation_option(prompt="데이터베이스를 덮어쓰게 됩니다. 계속하시겠습니까?")
def restore(file):
    import subprocess
    dsn = current_app.config["DATABASE_URL"]
    cmd = ["psql", dsn, "-f", file]
    subprocess.check_call(cmd)
```

## CLI와 테스트(자동화)

### pytest에서 CLI 실행 테스트

```python
# tests/test_cli_ops.py

def test_hello(runner):
    result = runner.invoke(args=["ops","hello","--name","Bob"])
    assert result.exit_code == 0
    assert "Hello, Bob!" in result.output
```

**conftest.py의 runner 픽스처**:

```python
@pytest.fixture()
def runner(app):
    return app.test_cli_runner()
```

### 시딩과 백필 작업 테스트

- 작은 테스트 전용 데이터베이스에 시딩 후 CLI 백필 실행 → 결과 검증
- 멱등성 확인: 동일 커맨드 2회 실행 → 결과 동일성 검증

```python
def test_seed_idempotent(runner, db):
    r1 = runner.invoke(args=["dbx","seed","--env","dev"])
    assert r1.exit_code == 0
    r2 = runner.invoke(args=["dbx","seed","--env","dev"])
    assert r2.exit_code == 0
```

## 운영 파이프라인에 통합

### GitHub Actions를 활용한 정기 작업

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
      - name: 유지보수 작업 실행
        env:
          FLASK_APP: "app:create_app('production')"
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: flask ops recompute-scores --yes
```
{% endraw %}

### 배포 훅에 마이그레이션과 시딩 통합

- Kubernetes 배포 전: `flask db upgrade` 실행(Job 형태)
- Kubernetes 배포 후: 필요시 `flask dbx seed --env=prod --no-force` 실행(멱등성 보장)

## 보안과 신뢰성 모범사례(CLI 관점)

- **최소 권한 원칙**: 운영 CLI는 읽기/쓰기 범위가 제한된 데이터베이스 계정 사용
- **비밀 정보 관리**: 환경변수/Secret만 사용, 코드나 저장소에 절대 저장 금지
- **장시간 작업 처리**: 타임아웃 설정/체크포인트/작업 재개 기능 지원
- **체계적 로깅**: JSON 구조화 로그 + `request_id` + 작업 지표(처리 건수/초당 처리량/오류 수)
- **알림 통합**: 중요한 결과/오류는 Slack/Webhook 연동
- **안전 장치**: 파괴적 작업은 반드시 `--dry-run` 기본값 + `--apply` 옵션 필요

**Slack 알림 간단 구현**

```python
def notify(text):
    import httpx, os
    url = os.getenv("SLACK_WEBHOOK_URL")
    if url:
        httpx.post(url, json={"text": text}, timeout=5)
```

## 흔한 안티패턴

- **create_app 미사용**: Flask 컨텍스트 없이 ORM 사용 → 설정/세션 누락
- **비멱등 시딩**: 매 실행마다 중복 생성/충돌 발생
- **대량 작업 단일 트랜잭션**: 롤백 시 문제, 락 장시간 유지
- **APScheduler를 다중 인스턴스 환경에서 무심코 실행**: 중복 작업 수행/경합 발생
- **표준 출력 대신 파일 로깅**: 컨테이너 환경에서 로그 수집 불가
- **예외 처리/종료 코드 무시**: CI/Kubernetes에서 실패 감지 불가

## 실전 적용 템플릿

### `app/cli/db.py` (요약)

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
    click.echo("완료")
```

### `app/cli/ops.py` (요약)

```python
import click
from app.extensions import db

@click.group("ops")
def ops_cli(): ...

@ops_cli.command("recompute")
@click.option("--apply", is_flag=True)
def recompute(apply):
    # ... 계산 로직
    if apply: db.session.commit()
    else: db.session.rollback()
```

### `app/cli/cron.py` (요약)

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

## 결론

이 부록에서는 Flask CLI를 실전 운영 도구로 활용하는 종합적인 방법을 다루었습니다. Click 기반의 커스텀 커맨드를 통해 운영과 유지보수 작업을 코드화하고, Flask-Migrate를 활용한 안전한 데이터베이스 스키마 변경과 멱등성 있는 데이터 시딩을 구현하는 방법을 살펴보았습니다. 또한 다양한 스케줄링 전략을 비교 분석하고, 실제 운영 환경에 적합한 접근 방식을 제시했습니다.

Flask CLI의 진정한 가치는 단순한 명령줄 인터페이스를 넘어, 애플리케이션의 모든 운영 작업을 일관된 방식으로 표준화하고 자동화하는 데 있습니다. 잘 설계된 CLI 시스템은 개발자의 생산성을 높이고, 운영 실수를 줄이며, 전체 시스템의 신뢰성을 향상시킵니다. 이러한 도구들을 프로젝트 초기부터 체계적으로 구축하고 지속적으로 개선해 나가는 것이 장기적인 프로젝트 성공의 핵심 요소가 될 것입니다.