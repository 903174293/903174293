#!/usr/bin/env bash
# ============================================================
# doocs -> Novordoc  一键搬运（全量 / 增量 自适应，可反复执行）
#
# 特性：
#   1. 自动判断模式
#        目标组织 0 个仓库  -> FULL 全量：把 doocs 全部仓库 fork 过来（全部分支）
#        目标组织已有仓库    -> INCREMENTAL 增量：
#                                · 只补 fork 缺失的新仓库
#                                · 已存在的 doocs fork 检查落后，自动快进同步
#                                · 冲突只报告，不覆盖
#   2. 幂等：想跑几次跑几次，已完成的自动跳过，不会重复建仓、不会报错
#   3. 结尾输出「全量分支核对表」，逐个仓库对比上游分支数
#
# 用法：
#   bash fork-doocs-to-novordoc.sh                 # 自适应执行
#   DRY_RUN=1 bash fork-doocs-to-novordoc.sh       # 只预览不动手
#   TARGET_ORG=别的组织名 bash fork-doocs-to-novordoc.sh
#   INCLUDE_SOURCE_FORKS=1 bash ...                # 连上游本身是二次 fork 的仓库也搬
#   CHECK_ONLY=1 bash ...                          # 只做分支核对，不 fork 不同步
#   SYNC_EXISTING=0 bash ...                       # 关掉增量同步，只补缺仓库
# ============================================================
set -uo pipefail

SOURCE_ORG="${SOURCE_ORG:-doocs}"
TARGET_ORG="${TARGET_ORG:-Novordoc}"
DRY_RUN="${DRY_RUN:-0}"
INCLUDE_SOURCE_FORKS="${INCLUDE_SOURCE_FORKS:-0}"
SYNC_EXISTING="${SYNC_EXISTING:-1}"       # 增量模式下是否对落后 fork 做快进同步
CHECK_ONLY="${CHECK_ONLY:-0}"
WAIT_AFTER_FORK="${WAIT_AFTER_FORK:-30}"  # 全量 fork 后等多少秒再做分支核对
MAX_RETRY="${MAX_RETRY:-3}"

LOG_DIR="fork-logs"; mkdir -p "$LOG_DIR"
STAMP="$(date +%Y%m%d-%H%M%S)"
RUN_LOG="$LOG_DIR/fork-$STAMP.log"
RESULT_FILE="$LOG_DIR/fork-result-$STAMP.tsv"

log() { printf '[%s] %s\n' "$(date +%H:%M:%S)" "$*" | tee -a "$RUN_LOG"; }
die() { printf '[ERROR] %s\n' "$*" | tee -a "$RUN_LOG" >&2; exit 1; }

# ---------- 0. 环境 ----------
command -v gh >/dev/null || die "未安装 gh"
command -v jq >/dev/null || die "未安装 jq"
gh auth status >/dev/null 2>&1 || die "gh 未登录：echo TOKEN | gh auth login --with-token"

# 带重试的 API 调用（api_retry <描述> <gh api 参数...>）
api_retry() {
  local desc="$1"; shift
  local attempt=1 resp code
  while [ "$attempt" -le "$MAX_RETRY" ]; do
    if resp=$(gh api "$@" 2>>"$RUN_LOG"); then printf '%s' "$resp"; return 0; fi
    code=$(printf '%s' "$resp" | grep -oE 'HTTP [0-9]{3}' | head -1)
    case "$code" in
      *429*|*5*)
        log "  ↻ $desc 受限($code)，$((attempt*20))s 后重试 ($attempt/$MAX_RETRY)"; sleep $((attempt*20)) ;;
      *) return 1 ;;
    esac
    attempt=$((attempt+1))
  done
  return 1
}

log "==================== 开始 $STAMP ===================="
[ "$DRY_RUN" = "1" ] && log "*** DRY RUN：只预览，不做任何写操作 ***"

# ---------- 1. 目标组织校验 ----------
log "检查目标组织 $TARGET_ORG ..."
ORG_LOGIN=$(gh api "orgs/$TARGET_ORG" --jq '.login' 2>/dev/null) || die "目标组织 $TARGET_ORG 不存在或当前账号无权限访问"
ORG_ID=$(gh api "orgs/$ORG_LOGIN" --jq '.id' 2>/dev/null)
log "目标组织确认：$ORG_LOGIN（org id=$ORG_ID，url: github.com/$ORG_LOGIN）"
[ "$ORG_LOGIN" != "$TARGET_ORG" ] && log "注意：输入名 $TARGET_ORG 与真实 login $ORG_LOGIN 不一致，以 $ORG_LOGIN 为准"

# ---------- 2. 两边清单 ----------
if [ "$INCLUDE_SOURCE_FORKS" = "1" ]; then
  SRC_JQ='.[] | .name'
else
  SRC_JQ='.[] | select(.fork == false) | .name'
fi

log "拉取 $SOURCE_ORG 仓库列表 ..."
mapfile -t SRC_REPOS < <(gh api --paginate "orgs/$SOURCE_ORG/repos?per_page=100" --jq "$SRC_JQ" 2>>"$RUN_LOG")
[ "${#SRC_REPOS[@]}" -gt 0 ] || die "上游仓库列表为空，检查组织名或网络"

log "读取 $ORG_LOGIN 现有仓库 ..."
mapfile -t DST_REPOS < <(gh api --paginate "orgs/$ORG_LOGIN/repos?per_page=100" --jq '.[].name' 2>>"$RUN_LOG")

printf '仓库\t动作\t结果\t落后提交\n' > "$RESULT_FILE"

in_dst() { local n="$1" e; for e in "${DST_REPOS[@]:-}"; do [ "$e" = "$n" ] && return 0; done; return 1; }

# ---------- 3. 模式判定 ----------
if [ "${#DST_REPOS[@]}" -eq 0 ] || [ "${FORCE_FULL:-0}" = "1" ]; then MODE="FULL"; else MODE="INCREMENTAL"; fi
log "模式：$MODE（上游 ${#SRC_REPOS[@]} 个 / 目标 ${#DST_REPOS[@]} 个）"

if [ "$MODE" = "FULL" ] && [ "$DRY_RUN" != "1" ]; then
  log "5 秒后开始执行，如组织名不对请立刻 Ctrl+C ..."
  sleep 5
fi

# ---------- 4. 阶段一：补 fork 缺失仓库 ----------
OK=0; SKIP=0; FAIL=0; SYNCED=0; UPTOD=0; CONF=0; NEWCNT=0

do_fork() {
  local repo="$1"
  if [ "$DRY_RUN" = "1" ]; then
    log "[DRY] fork $SOURCE_ORG/$repo -> $ORG_LOGIN/$repo（全部分支）"
    printf '%s\t%s\t%s\t-\n' "$repo" "fork" "dry-run" >> "$RESULT_FILE"; NEWCNT=$((NEWCNT+1)); return
  fi
  # -F = typed boolean，确保 default_branch_only=false 按布尔发送（= 全部分支）
  if api_retry "fork $repo" -X POST "repos/$SOURCE_ORG/$repo/forks" \
       -f organization="$ORG_LOGIN" -F default_branch_only=false >/dev/null; then
    log "✅ fork 成功 $repo（全部分支）"
    printf '%s\t%s\t%s\t-\n' "$repo" "fork" "forked" >> "$RESULT_FILE"
    OK=$((OK+1)); NEWCNT=$((NEWCNT+1)); sleep 2
  else
    log "✗ fork 失败 $repo（多为 token 缺组织权限，或同名仓库已存在）"
    printf '%s\t%s\t%s\t-\n' "$repo" "fork" "failed" >> "$RESULT_FILE"; FAIL=$((FAIL+1))
  fi
}

log "--- 阶段一：补 fork 缺失仓库 ---"
for repo in "${SRC_REPOS[@]}"; do
  if in_dst "$repo"; then
    SKIP=$((SKIP+1)); printf '%s\t%s\t%s\t-\n' "$repo" "fork" "skipped-existing" >> "$RESULT_FILE"; continue
  fi
  if [ "$CHECK_ONLY" = "1" ]; then log "跳过 $repo（CHECK_ONLY）"; continue; fi
  if [ "$DRY_RUN" = "1" ]; then
    log "[DRY] 缺失仓库待 fork：$repo（全部分支）"; NEWCNT=$((NEWCNT+1)); continue
  fi
  do_fork "$repo"
done
log "阶段一完成：新 fork $NEWCNT｜已存在跳过 $SKIP｜失败 $FAIL"

# ---------- 5. 阶段二：增量同步已存在的 doocs fork ----------
if [ "$SYNC_EXISTING" = "1" ] && [ "$CHECK_ONLY" != "1" ]; then
  log "--- 阶段二：检查并快进同步已有 fork ---"
  for repo in "${DST_REPOS[@]}"; do
    parent=$(gh api "repos/$ORG_LOGIN/$repo" --jq '.parent.full_name // ""' 2>>"$RUN_LOG") || { log "✗ 读不到 $repo"; FAIL=$((FAIL+1)); continue; }
    [ -n "$parent" ] || { log "· $repo 非 fork（你自己建的），跳过"; continue; }
    [ "${parent%%/*}" = "$SOURCE_ORG" ] || { log "· $repo 来自 $parent，非 $SOURCE_ORG，跳过"; continue; }

    branch=$(gh api "repos/$ORG_LOGIN/$repo" --jq '.default_branch' 2>>"$RUN_LOG")
    src_branch=$(gh api "repos/$SOURCE_ORG/$repo" --jq '.default_branch' 2>/dev/null) || src_branch="$branch"
    behind=$(gh api "repos/$SOURCE_ORG/$repo/compare/$src_branch...$ORG_LOGIN:$branch" --jq '.behind_by // 0' 2>/dev/null)
    case "$behind" in ''|*[!0-9]*) behind=0 ;; esac

    if [ "$behind" -le 0 ]; then
      UPTOD=$((UPTOD+1)); printf '%s\t%s\t%s\t%s\n' "$repo" "sync" "up-to-date" "0" >> "$RESULT_FILE"; continue
    fi
    if [ "$DRY_RUN" = "1" ]; then
      log "[DRY] $repo 落后 $behind 提交 → 待同步"
      printf '%s\t%s\t%s\t%s\n' "$repo" "sync" "behind-dry-run" "$behind" >> "$RESULT_FILE"; continue
    fi

    if out=$(api_retry "sync $repo" -X POST "repos/$ORG_LOGIN/$repo/merge-upstream" -f branch="$branch"); then
      log "↓ 同步完成 $repo（落后 $behind 提交）"
      printf '%s\t%s\t%s\t%s\n' "$repo" "sync" "synced" "$behind" >> "$RESULT_FILE"; SYNCED=$((SYNCED+1))
    elif printf '%s' "$out" | grep -qE 'conflict|409'; then
      log "⚠ 冲突 $repo（上游改了你改过的内容），需人工合并"
      printf '%s\t%s\t%s\t%s\n' "$repo" "sync" "conflict" "$behind" >> "$RESULT_FILE"; CONF=$((CONF+1))
    else
      log "✗ 同步失败 $repo：$(printf '%s' "${out:-}" | tail -2 | tr '\n' ' ')"
      printf '%s\t%s\t%s\t%s\n' "$repo" "sync" "failed" "$behind" >> "$RESULT_FILE"; FAIL=$((FAIL+1))
    fi
    sleep 1
  done
  log "阶段二完成：已同步 $SYNCED｜本来就新 $UPTOD｜冲突 $CONF"
fi

# ---------- 6. 阶段三：全量分支核对 ----------
log "--- 阶段三：全量分支核对（确认不是只 fork 了主分支）---"
if [ "$MODE" = "FULL" ] && [ "$NEWCNT" -gt 0 ] && [ "$DRY_RUN" != "1" ]; then
  log "fork 是异步的，等 ${WAIT_AFTER_FORK}s 让 GitHub 建仓完成 ..."
  sleep "$WAIT_AFTER_FORK"
fi

CHK_OK=0; CHK_GAP=0; CHK_MISS=0
printf '%-32s %-8s %-6s %-8s %s\n' "仓库" "上游分支" "你的" "状态" "备注" | tee -a "$RUN_LOG"
for repo in "${SRC_REPOS[@]}"; do
  a=$(gh api "repos/$SOURCE_ORG/$repo/branches" --paginate --jq 'length' 2>/dev/null || echo 0)
  b=$(gh api "repos/$ORG_LOGIN/$repo/branches" --paginate --jq 'length' 2>/dev/null || echo 0)
  case "$a" in ''|*[!0-9]*) a=0 ;; esac
  case "$b" in ''|*[!0-9]*) b=0 ;; esac

  if [ "$b" -eq 0 ]; then st="MISSING"; CHK_MISS=$((CHK_MISS+1)); note="仓库还没出现/仍在建"
  elif [ "$b" -eq "$a" ]; then st="OK"; CHK_OK=$((CHK_OK+1)); note=""
  elif [ "$b" -lt "$a" ]; then st="GAP"; CHK_GAP=$((CHK_GAP+1)); note="少 $((a-b)) 个分支"
  else st="EXTRA"; note="你这边分支更多（自己加的？）"
  fi
  printf '%-32s %-8s %-6s %-8s %s\n' "$repo" "$a" "$b" "$st" "$note" | tee -a "$RUN_LOG"
done

# ---------- 7. 汇总 ----------
echo | tee -a "$RUN_LOG"
log "==================== 汇总 ===================="
log "模式 $MODE｜新 fork $NEWCNT｜同步 $SYNCED｜已最新 $UPTOD｜冲突 $CONF｜失败 $FAIL"
log "分支核对：全对 $CHK_OK｜分支缺口 $CHK_GAP｜未就绪 $CHK_MISS"
log "明细文件：$RESULT_FILE"
if [ "$CHK_GAP" -gt 0 ] || [ "$CHK_MISS" -gt 0 ]; then
  log "GAP/MISSING 先等 5~10 分钟，再跑：CHECK_ONLY=1 bash $(basename "$0")"
  log "若仍是 GAP，用 git 补齐（REPO 换成仓库名）："
  log "  TOKEN=\$(gh auth token) && REPO=缺口仓库名"
  log "  git clone --bare https://x-access-token:\$TOKEN@github.com/$SOURCE_ORG/\$REPO.git /tmp/\$REPO.git"
  log "  cd /tmp/\$REPO.git && git push --mirror https://x-access-token:\$TOKEN@github.com/$ORG_LOGIN/\$REPO.git"
fi
[ "$CONF" -gt 0 ] && log "冲突仓库：$(awk -F'\t' '$3=="conflict"{printf "%s ", $1}' "$RESULT_FILE")"
log "==================== 结束 ===================="
exit 0
