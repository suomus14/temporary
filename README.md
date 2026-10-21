#!/usr/bin/env python3

# 必要な標準ライブラリを読み込む。
import glob
import os
import re
import subprocess
import sys
from datetime import datetime

# 装置情報の取得先を固定する。
ESPRITSET_PATH = "/console/mdas/MachineList"
# 装置情報ファイルの名前を固定する。
ESPRITSET_NAME = "*.espritset"
# 起動時PID情報ファイルの検索先を固定する。
PID_LIST_GLOB = "/var/tmp/*/pid_list/pid_*.espritset"
# 対象にする親プロセス名を固定する。
PARENT_PROCESS_NAME = "time_handler.py"
# 対象にする親プロセスのハンドラー名を固定する。
PARENT_HANDLER_NAME = "DataGenerator"
# 対象にする子プロセス名を固定する。
CHILD_PROCESS_PATTERN = "executor_DataGenerator[.]py"
# 再起動に使うMDAS起動スクリプトを固定する。
MDAS_ACTIVATION_SCRIPT = "/home/ifour/TOOLS/mdas_activation.sh"
# ログ出力先のベースパスを固定する。
LOG_FILE_PATH = "/var/log/mdas/check_duplicate.log"


# 重複起動を確認する。
class DuplicateChecker:
    # ログファイルを開いて状態を初期化する。
    def __init__(self):
        # ログ出力先のディレクトリを取得する。
        log_directory = os.path.dirname(LOG_FILE_PATH)
        # ログ出力先のディレクトリが指定されている場合に作成する。
        if log_directory:
            # 存在しないログ出力先のディレクトリを作成する。
            os.makedirs(log_directory, exist_ok=True)
        # ベースパスからファイル名と拡張子を分離する。
        log_base, log_extension = os.path.splitext(LOG_FILE_PATH)
        # 日付をファイル名に付けた出力パスを作成する。
        dated_log_path = "{}_{}{}".format(
            log_base,
            datetime.now().strftime("%Y%m%d"),
            log_extension,
        )
        # 当日分のログファイルを追記モードで開く。
        self._log_file = open(dated_log_path, "a", encoding="utf-8")
        # 重複検出状態を初期化する。
        self._duplicate_found = False
        # エラー発生状態を初期化する。
        self._error_found = False

    # 形式をそろえてログを出力する。
    def _log(self, level, message):
        # 現在時刻をログ形式に変換する。
        timestamp = datetime.now().strftime("%Y/%m/%d %H:%M:%S")
        # ログ行を生成する。
        log_line = f"{timestamp} [{level}] {message}"
        # ログ行を標準出力に表示する。
        print(log_line)
        # ログ行をファイルに書き込む。
        self._log_file.write(log_line + "\n")
        # ログ行をすぐに保存する。
        self._log_file.flush()

    # 情報ログを出力する。
    def log_info(self, message):
        # INFOレベルでログを出力する。
        self._log("INFO ", message)

    # 警告ログを出力する。
    def log_warn(self, message):
        # WARNレベルでログを出力する。
        self._log("WARN ", message)

    # エラーログを出力する。
    def log_error(self, message):
        # エラー発生状態を記録する。
        self._error_found = True
        # ERRORレベルでログを出力する。
        self._log("ERROR", message)

    # MachineListから装置情報を取得する。
    def get_equip_info(self):
        # 装置情報の取得開始を記録する。
        self.log_info(
            "==================== Read the espritset files ==========================="
        )
        # MachineListから装置情報ファイルを検索する。
        file_list = sorted(glob.glob(os.path.join(ESPRITSET_PATH, ESPRITSET_NAME)))
        # 検索した装置情報ファイル数を記録する。
        self.log_info(f"Number of espritset files = {len(file_list)}")
        # 装置情報ファイルがない場合にエラーを記録する。
        if not file_list:
            # 装置情報がないことをログに出力する。
            self.log_error("Espritset file not found")
        # 取得した装置情報を格納する。
        equip_list = []
        # 装置情報ファイルを順に読み込む。
        for file_path in file_list:
            # 装置情報の値を初期化する。
            equip_info = {"tool_id": "", "serial_num": "", "machine_type": ""}
            # 装置情報ファイルを読み込む。
            try:
                # UTF-8のテキストとして装置情報を開く。
                with open(file_path, "r", encoding="utf-8") as equip_file:
                    # 装置情報の各行を確認する。
                    for line in equip_file:
                        # TOOL_IDの値を取得する。
                        tool_id_match = re.match(
                            r"^\s*setenv\s+TOOL_ID\s+(.+?)\s*$", line
                        )
                        # シリアル番号の値を取得する。
                        serial_match = re.match(
                            r"^\s*setenv\s+SERIAL_NUM\s+(.+?)\s*$", line
                        )
                        # 装置種別の値を取得する。
                        type_match = re.match(
                            r"^\s*setenv\s+ESPRIT_MACHINE_TYPE\s+(.+?)\s*$", line
                        )
                        # TOOL_IDが見つかった場合に保存する。
                        if tool_id_match:
                            # 引用符を除いてTOOL_IDを保存する。
                            equip_info["tool_id"] = tool_id_match.group(1).strip("\"'")
                        # シリアル番号が見つかった場合に保存する。
                        if serial_match:
                            # 引用符を除いてシリアル番号を保存する。
                            equip_info["serial_num"] = serial_match.group(1).strip(
                                "\"'"
                            )
                        # 装置種別が見つかった場合に保存する。
                        if type_match:
                            # 引用符を除いて装置種別を保存する。
                            equip_info["machine_type"] = type_match.group(1).strip(
                                "\"'"
                            )
            # ファイルを読み込めない場合にエラーを記録する。
            except (OSError, UnicodeError) as error:
                # 読み込みに失敗したファイル名と理由を記録する。
                self.log_error(f"Cannot read {file_path}: {error}")
                # 次の装置情報ファイルを確認する。
                continue
            # 読み込んだ装置情報ファイル名を記録する。
            self.log_info(f"- {os.path.basename(file_path)}")
            # TOOL_IDを記録する。
            self.log_info("    Name: {}".format(equip_info["tool_id"]))
            # 装置種別を記録する。
            self.log_info("    Type: {}".format(equip_info["machine_type"]))
            # シリアル番号を記録する。
            self.log_info("    S/N : {}".format(equip_info["serial_num"]))
            # TOOL_IDがない場合に装置を対象外にする。
            if not equip_info["tool_id"]:
                # PIDファイルを特定できない理由を記録する。
                self.log_error(f"TOOL_ID not found in {file_path}")
                # 次の装置情報ファイルを確認する。
                continue
            # 取得した装置情報を一覧に追加する。
            equip_list.append(equip_info)
        # 取得した装置情報一覧を返す。
        return equip_list

    # pid_listからDataGeneratorの親プロセスPIDを取得する。
    def get_parent_pids(self, tool_id):
        # 対象装置のPIDファイル名を決定する。
        expected_file_name = f"pid_{tool_id}.espritset"
        # 対象装置のPIDファイルを検索する。
        pid_files = [
            path
            for path in glob.glob(PID_LIST_GLOB)
            if os.path.basename(path) == expected_file_name
        ]
        # 親プロセスPIDを重複なく格納する。
        parent_pids = set()
        # 装置別PIDファイルを順に読み込む。
        for pid_file_path in pid_files:
            # PIDファイルを読み込む。
            try:
                # UTF-8のテキストとしてPIDファイルを開く。
                with open(pid_file_path, "r", encoding="utf-8") as pid_file:
                    # PIDファイルの各行を確認する。
                    for line in pid_file:
                        # 行を空白区切りで分割する。
                        fields = line.split()
                        # DataGenerator用time_handlerの行だけを処理する。
                        if (
                            len(fields) >= 3
                            and fields[0] == PARENT_PROCESS_NAME
                            and fields[2] == PARENT_HANDLER_NAME
                        ):
                            # PIDを整数として読み取る。
                            try:
                                # 親プロセスPIDを重複なく保存する。
                                parent_pids.add(int(fields[1]))
                            # PIDが数値でない場合に警告する。
                            except ValueError:
                                # 不正なPID情報をログに出力する。
                                self.log_warn(
                                    f"Invalid parent PID in {pid_file_path}: {line.strip()}"
                                )
            # PIDファイルを読み込めない場合にエラーを記録する。
            except (OSError, UnicodeError) as error:
                # 読み込みに失敗したファイル名と理由を記録する。
                self.log_error(f"Cannot read {pid_file_path}: {error}")
        # PIDファイルが見つからない場合に警告する。
        if not pid_files:
            # 対象装置のPIDファイルがないことを記録する。
            self.log_error(f"PID list file not found for TOOL_ID {tool_id}")
        # 親プロセスが見つからない場合に警告する。
        elif not parent_pids:
            # DataGeneratorの親PIDがないことを記録する。
            self.log_warn(f"DataGenerator parent PID not found for TOOL_ID {tool_id}")
        # 親プロセスPIDを昇順で返す。
        return sorted(parent_pids)

    # pgrepで親プロセスに属するDataGeneratorの子PIDを取得する。
    def get_child_pids(self, parent_pid):
        # pgrepの親PID指定とプロセス名指定を組み立てる。
        command = ["pgrep", "-P", str(parent_pid), "-f", CHILD_PROCESS_PATTERN]
        # pgrepを実行して子PIDを取得する。
        try:
            # pgrepの標準出力と標準エラーを取得する。
            result = subprocess.run(
                command,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
                check=False,
            )
        # pgrepを起動できない場合にエラーを記録する。
        except OSError as error:
            # pgrepの実行失敗をログに出力する。
            self.log_error(f"Cannot execute pgrep: {error}")
            # 取得できないことを呼び出し元へ伝える。
            return None
        # pgrepが該当プロセスなしを返した場合に空一覧を返す。
        if result.returncode == 1:
            # 子プロセスがないことを空一覧で返す。
            return []
        # pgrepが異常終了した場合にエラーを記録する。
        if result.returncode != 0:
            # pgrepの標準エラーをログに出力する。
            self.log_error(
                f"pgrep failed for parent PID {parent_pid}: {result.stderr.strip()}"
            )
            # 取得できないことを呼び出し元へ伝える。
            return None
        # pgrepの出力から子PID一覧を作成する。
        child_pids = result.stdout.splitlines()
        # pgrepのPID出力が数値か確認する。
        if any(not child_pid.isdigit() for child_pid in child_pids):
            # 不正なpgrep出力をエラーとして記録する。
            self.log_error(
                f"Invalid pgrep output for parent PID {parent_pid}: {result.stdout.strip()}"
            )
            # 不正な出力を呼び出し元へ伝える。
            return None
        # 子PID一覧を返す。
        return child_pids

    # 装置ごとにDataGeneratorの子プロセス数を確認する。
    def check_equip(self, equip_info):
        # 装置識別情報をログに出力する。
        self.log_info(
            "- Name: {} / Type: {} / S/N: {}".format(
                equip_info["tool_id"],
                equip_info["machine_type"],
                equip_info["serial_num"],
            )
        )
        # pid_listから対象装置の親PID一覧を取得する。
        parent_pids = self.get_parent_pids(equip_info["tool_id"])
        # 装置に紐づく子PIDを格納する。
        child_pids = set()
        # 親プロセスごとに子プロセス数を確認する。
        for parent_pid in parent_pids:
            # 親プロセスの子PID一覧をpgrepで取得する。
            parent_children = self.get_child_pids(parent_pid)
            # pgrepが失敗した場合に次の親プロセスへ進む。
            if parent_children is None:
                # 次の親プロセスを確認する。
                continue
            # 親PIDごとの子プロセス数を記録する。
            self.log_info(
                f"    Parent PID: {parent_pid} / DataGenerator child count: {len(parent_children)}"
            )
            # 取得した子PIDを装置の一覧に追加する。
            child_pids.update(parent_children)
        # 子プロセスが複数ある場合に重複を記録する。
        if len(child_pids) > 1:
            # 重複状態を記録する。
            self._duplicate_found = True
            # 重複した装置と子PID一覧をエラー出力する。
            self.log_error(
                "DUPLICATE DataGenerator processes: TOOL_ID={} count={} child PIDs={}".format(
                    equip_info["tool_id"],
                    len(child_pids),
                    ",".join(sorted(child_pids, key=int)),
                )
            )
        # 子プロセスが1つ以下の場合に確認結果を記録する。
        else:
            # 子PIDがない場合に正常状態を記録する。
            self.log_info(f"    DataGenerator child count: {len(child_pids)}")

    # 重複が発生した場合に全装置のMDASを再起動する。
    def restart_all_mdas(self):
        # 全装置を対象に非対話モードで再起動するコマンドを組み立てる。
        command = [MDAS_ACTIVATION_SCRIPT, "-a", "-y"]
        # 全装置の再起動開始をログに出力する。
        self.log_warn("Restarting MDAS for all equipment using mdas_activation.sh")
        # MDAS起動スクリプトを実行する。
        try:
            # 標準出力と標準エラーを取得して起動結果を確認する。
            result = subprocess.run(
                command,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
                check=False,
            )
        # activationスクリプトを実行できない場合にエラーを記録する。
        except OSError as error:
            # 全装置の再起動コマンド失敗をログに出力する。
            self.log_error(f"Cannot restart all MDAS equipment: {error}")
            # 再起動処理を終了する。
            return
        # 起動スクリプトの標準出力をログに記録する。
        if result.stdout.strip():
            # 標準出力をINFOログに出力する。
            self.log_info(result.stdout.strip())
        # 起動スクリプトの標準エラーをログに記録する。
        if result.stderr.strip():
            # 標準エラーをWARNログに出力する。
            self.log_warn(result.stderr.strip())
        # 再起動に失敗した場合にエラーを記録する。
        if result.returncode != 0:
            # 終了コードを含めて再起動失敗を記録する。
            self.log_error(
                f"MDAS restart failed for all equipment with exit code {result.returncode}"
            )
            # 再起動成功処理へ進まない。
            return
        # 再起動成功をログに記録する。
        self.log_info("MDAS restart completed for all equipment")

    # 全装置の確認を実行する。
    def main(self):
        # 重複確認の開始をログに出力する。
        self.log_info(
            "==================== Check DataGenerator duplicates ===================="
        )
        # MachineListから全装置情報を取得する。
        equip_list = self.get_equip_info()
        # 装置ごとに重複確認を実行する。
        for equip_info in equip_list:
            # 対象装置の子プロセス数を確認する。
            self.check_equip(equip_info)
        # 重複が見つかった場合に全装置のMDASを再起動する。
        if self._duplicate_found:
            # 全装置のMDAS再起動を実行する。
            self.restart_all_mdas()
            # 重複状態を終了コード1で返す。
            return 1
        # エラーが発生した場合にエラー終了コードを返す。
        if self._error_found:
            # エラー状態を終了コード2で返す。
            return 2
        # 問題がない場合に正常終了コードを返す。
        return 0

    # ログファイルを閉じて確認を終了する。
    def close(self):
        # ログファイルを閉じる。
        self._log_file.close()


# 直接実行された場合に確認処理を開始する。
if __name__ == "__main__":
    # 確認処理インスタンスを生成する。
    checker = DuplicateChecker()
    # 確認処理後に必ずログファイルを閉じる。
    try:
        # 確認結果を取得する。
        exit_code = checker.main()
    # 例外発生時もログファイルを閉じる。
    finally:
        # ログファイルを閉じる。
        checker.close()
    # 確認結果を終了コードとして返す。
    sys.exit(exit_code)





－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－---------

#!/usr/bin/env python3
"""
MDASサーバーのI/O停滞調査用プローブ（Python 3標準ライブラリのみ）。

目的
====
DataGeneratorの重複はI/O停滞の二次被害である可能性があるため、
重複検知時だけでなく平常時から、プロセス単位のI/Oカウンターと
ホスト／ブロックデバイスの累積カウンターを同じ時刻軸で記録する。
出力先は /var/log/freeze_probe。10秒ごとに /proc を読み、日付別TSVへ追記する。

AI解析時の観点
==============
1. process-*.tsv の同一 pid + start_ticks を1プロセスとして追跡する。
   PIDだけでは終了後に再利用される可能性がある。
2. event が io_delta の行を調べ、delta_read_bytes / delta_write_bytes
   が大きいプロセスとその時間帯を特定する。rchar / wchar はキャッシュ経由を
   含むため、実ストレージI/O量の目安は read_bytes / write_bytes を優先する。
   初回観測値は全プロセス分を出力せず、初回観測がD状態の場合だけ
   blocked_D行に累積値を記録する。io_delta行は以降のカウンター差分を表す。
3. event=blocked_D は、そのサンプル時点で割り込み不能状態(D)だったことを示す。
   wchan と delayacct_blkio_ticks の変化を参照する。D状態だけでディスク待ちと
   断定せず、pressure-io、device、kernelログと照合する。
4. device-*.tsv はデバイス単位の累積カウンター。隣接サンプルの差をとり、
   I/O量、完了要求数、in_flight、処理時間の増加を比較する。
   /proc/diskstatsのセクターは通常512バイト。デバイスマッパー等の階層を
   またいで値を単純合算すると同じI/Oを重複計上する場合がある。
5. host-*.tsv の pressure-io はI/O待ちの影響、procs_blockedはブロック中数、
   meminfo/vmstatはメモリ回収・スワップ・writeback等を確認する。
   同一時刻近辺でプロセス差分、デバイス差分、I/O圧力、メモリ状況を突き合わせる。
6. PIDごとのI/Oバイト数はデバイス占有時間や要求ごとの遅延ではない。
   writeback、swap、RAID/multipath、NFS/SAN/仮想基盤などでは発行元が一対一に
   対応しない場合がある。ホスト記録で説明できなければ、それらの層の同時刻の
   ログ・統計も必要。短命プロセスは10秒ポーリングの間に終了すると見逃し得る。

運用上の注意
============
rootで起動する（/proc/<PID>/io等の権限と網羅性のため）。コマンドラインや
環境変数は秘密情報を含み得るため記録しない。/procの読み取りは通常ディスク
読み取りではないが、プロセス数に応じたCPU負荷と、TSV追記のディスク書き込みは
発生する。強制fsyncや全ログの毎回出力は行わず、収集間隔は10秒とする。

保存先が障害対象と同じディスクの場合、ディスク停滞中はログ書き込みも遅れたり
失敗したりする。この制約はスクリプトでは解消できない。ログ容量を定期確認し、
運用側で保管期間を決めて古い日付ファイルを削除する。障害調査中のファイルは
解析・回収が終わるまで削除しない。実運用前にテスト環境で負荷と出力量を確認する。

ログ
====
host-YYYYMMDD.tsv:
  timestamp, monotonic_seconds, source, key, value
  ホスト状態の選択したカウンター。カウンター系は差分で読む。
device-YYYYMMDD.tsv:
  timestamp, monotonic_seconds, major, minor, device, fields
  fieldsは /proc/diskstats のデバイス名以降の数値列を順番通り保存する。
  Linuxカーネルにより列数が異なることがあるため、値の意味は上記と
  AlmaLinux 9.7のカーネルが提供する /proc/diskstats の定義に沿って解釈する。
process-YYYYMMDD.tsv:
  timestamp, monotonic_seconds, pid, start_ticks, comm, exe, ppid, state,
  wchan, event, delta/cumulative I/Oカウンター, blkio_ticks
  イベント行のみ出力する。初回観測値は通常出力しないため、
  監視開始前からの累積値を差分と誤認することがない。
errors-YYYYMMDD.tsv:
  読み取り権限や/procエントリ消失など、収集時の問題。
"""

import datetime
import os
import sys
import time


OUTPUT_DIR = "/var/log/freeze_probe"
INTERVAL_SECONDS = 10

PROCESS_COUNTERS = (
    "rchar",
    "wchar",
    "read_bytes",
    "write_bytes",
    "syscr",
    "syscw",
    "cancelled_write_bytes",
)

MEMINFO_KEYS = (
    "MemTotal",
    "MemFree",
    "MemAvailable",
    "Buffers",
    "Cached",
    "SwapCached",
    "Active(file)",
    "Inactive(file)",
    "Dirty",
    "Writeback",
    "Slab",
    "SReclaimable",
    "SwapTotal",
    "SwapFree",
)

VMSTAT_KEYS = (
    "pswpin",
    "pswpout",
    "pgmajfault",
    "pgscan_kswapd",
    "pgscan_direct",
    "pgsteal_kswapd",
    "pgsteal_direct",
    "allocstall",
    "allocstall_dma",
    "allocstall_dma32",
    "allocstall_normal",
    "allocstall_movable",
    "nr_dirty",
    "nr_writeback",
    "oom_kill",
)

HOST_HEADER = "timestamp\tmonotonic_seconds\tsource\tkey\tvalue\n"
DEVICE_HEADER = (
    "timestamp\tmonotonic_seconds\tmajor\tminor\tdevice\tfields\n"
)
PROCESS_HEADER = (
    "timestamp\tmonotonic_seconds\tpid\tstart_ticks\tcomm\texe\tppid\t"
    "state\twchan\tevent\tdelta_rchar\tdelta_wchar\tdelta_read_bytes\t"
    "delta_write_bytes\tdelta_syscr\tdelta_syscw\t"
    "delta_cancelled_write_bytes\trchar\twchar\tread_bytes\twrite_bytes\t"
    "syscr\tsyscw\tcancelled_write_bytes\tblkio_ticks\tdelta_blkio_ticks\n"
)
ERROR_HEADER = "timestamp\tmonotonic_seconds\tsource\terror\n"


def timestamp():
    """ローカルタイムゾーン付き時刻を秒精度で返す。"""
    return datetime.datetime.now().astimezone().isoformat(timespec="seconds")


def current_day():
    return datetime.datetime.now().strftime("%Y%m%d")


def tsv(value):
    """TSVの列を壊さないようタブ・改行を置換する。"""
    return str(value).replace("\t", " ").replace("\r", " ").replace("\n", " ")


def append_batch(filename, header, rows):
    """一回のサンプル分をまとめて追記し、毎行の書き込みを避ける。"""
    if not rows:
        return
    path = os.path.join(OUTPUT_DIR, filename)
    new_file = not os.path.exists(path)
    with open(path, "a", encoding="utf-8") as output:
        if new_file:
            output.write(header)
        output.writelines(rows)


def read_proc_file(path):
    with open(path, "r", encoding="utf-8", errors="replace") as source:
        return source.read()


def parse_key_values(path):
    values = {}
    for line in read_proc_file(path).splitlines():
        fields = line.split()
        if len(fields) >= 2:
            values[fields[0].rstrip(":")] = fields[1]
    return values


def record_error(errors, ts, mono, source, error):
    errors.append(
        "\t".join((ts, f"{mono:.3f}", tsv(source), tsv(error))) + "\n"
    )


def collect_host(ts, mono, errors):
    rows = []

    def add(source, key, value):
        rows.append(
            "\t".join((ts, f"{mono:.3f}", source, tsv(key), tsv(value))) + "\n"
        )

    # CPU累積値、実行中・ブロック中プロセス数、コンテキスト切替等。
    try:
        for line in read_proc_file("/proc/stat").splitlines():
            parts = line.split()
            if parts and (
                parts[0].startswith("cpu")
                or parts[0] in ("procs_running", "procs_blocked", "ctxt", "processes")
            ):
                add("stat", parts[0], " ".join(parts[1:]))
    except OSError as error:
        record_error(errors, ts, mono, "/proc/stat", error)

    # メモリ、ページ回収、スワップに関連する値のみを記録してログ量を抑える。
    for source, path, selected in (
        ("meminfo", "/proc/meminfo", set(MEMINFO_KEYS)),
        ("vmstat", "/proc/vmstat", set(VMSTAT_KEYS)),
    ):
        try:
            values = parse_key_values(path)
            for key in selected:
                if key in values:
                    add(source, key, values[key])
        except OSError as error:
            record_error(errors, ts, mono, path, error)

    # PSIが利用可能なカーネルでは、I/O待ちで失われた時間の割合を記録する。
    try:
        for line in read_proc_file("/proc/pressure/io").splitlines():
            parts = line.split()
            if parts:
                add("pressure-io", parts[0], " ".join(parts[1:]))
    except FileNotFoundError:
        add("pressure-io", "availability", "unavailable")
    except OSError as error:
        record_error(errors, ts, mono, "/proc/pressure/io", error)

    # ロード平均と実行・総プロセス数を記録する。
    try:
        add("loadavg", "values", read_proc_file("/proc/loadavg").strip())
    except OSError as error:
        record_error(errors, ts, mono, "/proc/loadavg", error)

    return rows


def collect_devices(ts, mono, errors):
    rows = []
    try:
        for line in read_proc_file("/proc/diskstats").splitlines():
            fields = line.split()
            if len(fields) >= 4:
                major, minor, device = fields[:3]
                counters = fields[3:]
                rows.append(
                    "\t".join(
                        (
                            ts,
                            f"{mono:.3f}",
                            major,
                            minor,
                            tsv(device),
                            " ".join(counters),
                        )
                    )
                    + "\n"
                )
    except OSError as error:
        record_error(errors, ts, mono, "/proc/diskstats", error)
    return rows


def parse_process_stat(text):
    """comm内の空白や括弧を考慮して /proc/PID/stat を解析する。"""
    close = text.rfind(")")
    if close < 0:
        raise ValueError("malformed process stat")
    comm = text[text.find("(") + 1 : close]
    fields = text[close + 2 :].split()
    if len(fields) <= 19:
        raise ValueError("process stat has too few fields")
    state = fields[0]
    ppid = fields[1]
    start_ticks = fields[19]  # field 22 (starttime); fields starts at field 3.
    blkio_ticks = fields[39] if len(fields) > 39 else ""
    return comm, state, ppid, start_ticks, blkio_ticks


def read_process(proc_name):
    base = os.path.join("/proc", proc_name)
    comm, state, ppid, start_ticks, blkio_ticks = parse_process_stat(
        read_proc_file(os.path.join(base, "stat"))
    )

    io_values = {}
    for line in read_proc_file(os.path.join(base, "io")).splitlines():
        key, value = line.split(":", 1)
        io_values[key] = int(value.strip())

    try:
        exe = os.readlink(os.path.join(base, "exe"))
    except OSError:
        exe = ""
    try:
        wchan = read_proc_file(os.path.join(base, "wchan")).strip()
    except OSError:
        wchan = ""

    return {
        "pid": int(proc_name),
        "comm": comm,
        "exe": exe,
        "ppid": ppid,
        "state": state,
        "start_ticks": start_ticks,
        "blkio_ticks": blkio_ticks,
        "wchan": wchan,
        "io": io_values,
    }


def process_row(ts, mono, process, event, deltas=None):
    deltas = deltas or {}
    io_values = process["io"]
    fields = [
        ts,
        f"{mono:.3f}",
        process["pid"],
        process["start_ticks"],
        tsv(process["comm"]),
        tsv(process["exe"]),
        process["ppid"],
        process["state"],
        tsv(process["wchan"]),
        event,
    ]
    fields.extend(deltas.get(key, "") for key in PROCESS_COUNTERS)
    fields.extend(io_values.get(key, "") for key in PROCESS_COUNTERS)
    fields.append(process["blkio_ticks"])
    fields.append(deltas.get("blkio_ticks", ""))
    return "\t".join(map(str, fields)) + "\n"


def collect_processes(ts, mono, previous, errors):
    current = {}
    rows = []

    try:
        entries = os.listdir("/proc")
    except OSError as error:
        record_error(errors, ts, mono, "/proc", error)
        return current, rows

    for proc_name in entries:
        if not proc_name.isdigit():
            continue
        try:
            process = read_process(proc_name)
        except (FileNotFoundError, ProcessLookupError):
            # プロセス終了と採取が競合した場合は、次回の消失判定に任せる。
            continue
        except PermissionError as error:
            record_error(errors, ts, mono, f"/proc/{proc_name}", error)
            continue
        except (OSError, ValueError, IndexError) as error:
            record_error(errors, ts, mono, f"/proc/{proc_name}", error)
            continue

        identity = (process["pid"], process["start_ticks"])
        current[identity] = process
        old = previous.get(identity)

        if old is None:
            # 初回値は監視開始前からの累積かもしれないため、差分扱いしない。
            if process["state"] == "D":
                rows.append(process_row(ts, mono, process, "blocked_D"))
            continue

        deltas = {
            key: process["io"].get(key, 0) - old["io"].get(key, 0)
            for key in PROCESS_COUNTERS
        }
        delta_ticks = ""
        if process["blkio_ticks"] and old["blkio_ticks"]:
            try:
                delta_ticks = int(process["blkio_ticks"]) - int(old["blkio_ticks"])
            except ValueError:
                delta_ticks = ""
        deltas["blkio_ticks"] = delta_ticks

        if any(value != 0 for value in deltas.values()):
            rows.append(process_row(ts, mono, process, "io_delta", deltas))
        if process["state"] == "D":
            # D状態が続いたサンプルを残し、継続時間の下限を把握できるようにする。
            rows.append(process_row(ts, mono, process, "blocked_D", deltas))
        elif process["state"] != old["state"]:
            rows.append(process_row(ts, mono, process, "state_change", deltas))

    # 終了したプロセスの最終累積値を、正しい列位置で記録する。
    # 終了が観測された時刻は実際の終了時刻ではなく、前回採取以降の範囲である。
    for identity, process in previous.items():
        if identity not in current:
            rows.append(process_row(ts, mono, process, "exit_observed"))

    return current, rows


def write_metadata():
    path = os.path.join(OUTPUT_DIR, "metadata.txt")
    if os.path.exists(path):
        return
    try:
        clk_tck = os.sysconf("SC_CLK_TCK")
    except (ValueError, OSError):
        clk_tck = "unavailable"
    try:
        delayacct = read_proc_file("/proc/sys/kernel/task_delayacct").strip()
    except OSError:
        delayacct = "unavailable"

    details = [
        "Freeze probe metadata",
        f"started_at={timestamp()}",
        f"hostname={os.uname().nodename}",
        f"kernel={os.uname().release}",
        f"python={sys.version.split()[0]}",
        f"interval_seconds={INTERVAL_SECONDS}",
        f"clk_tck={clk_tck}",
        f"task_delayacct={delayacct}",
        "Run as root to maximize /proc process I/O visibility.",
        "No process command lines or environment variables are collected.",
    ]
    with open(path, "x", encoding="utf-8") as output:
        output.write("\n".join(details) + "\n")


def main():
    os.makedirs(OUTPUT_DIR, mode=0o700, exist_ok=True)
    os.umask(0o077)
    write_metadata()
    previous = {}

    while True:
        cycle_start = time.monotonic()
        ts = timestamp()
        mono = time.monotonic()
        errors = []

        host_rows = collect_host(ts, mono, errors)
        device_rows = collect_devices(ts, mono, errors)
        current, process_rows = collect_processes(
            ts, mono, previous, errors
        )

        day = current_day()
        append_batch(f"host-{day}.tsv", HOST_HEADER, host_rows)
        append_batch(f"device-{day}.tsv", DEVICE_HEADER, device_rows)
        append_batch(f"process-{day}.tsv", PROCESS_HEADER, process_rows)
        append_batch(
            f"errors-{day}.tsv", ERROR_HEADER, errors
        )

        previous = current
        elapsed = time.monotonic() - cycle_start
        time.sleep(max(0, INTERVAL_SECONDS - elapsed))


if __name__ == "__main__":
    main()














－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－---------




'''
---------------------------------------------
Copyright(C) Canon Inc. All right reserved.
---------------------------------------------
'''
import socket
import select
import json
import os
import loggerForEsprit

POLLING_TIMEOUT_ENV = 'MDAS_REQUE_RESPONSE_TIMEOUT_SEC'
DEFAULT_POLLING_TIMEOUT_SEC = 20
MIN_POLLING_TIMEOUT_SEC = 1
MAX_POLLING_TIMEOUT_SEC = 3600

class ScriptSender():
    INT32_BYTES = 4

    def __init__(self, service):
        self.port = socket.getservbyname(service, 'tcp')
        self.logger = loggerForEsprit.getLogger()
        timeout_sec = os.environ.get(POLLING_TIMEOUT_ENV)
        if timeout_sec is None:
            self.logger.info(
                '%s is not set; using default timeout of %d seconds',
                POLLING_TIMEOUT_ENV,
                DEFAULT_POLLING_TIMEOUT_SEC,
            )
            timeout_sec = DEFAULT_POLLING_TIMEOUT_SEC
        else:
            try:
                timeout_sec = int(timeout_sec)
            except ValueError:
                timeout_sec = None

            if timeout_sec is None or not MIN_POLLING_TIMEOUT_SEC <= timeout_sec <= MAX_POLLING_TIMEOUT_SEC:
                self.logger.warning(
                    'Invalid %s value; using default timeout of %d seconds',
                    POLLING_TIMEOUT_ENV,
                    DEFAULT_POLLING_TIMEOUT_SEC,
                )
                timeout_sec = DEFAULT_POLLING_TIMEOUT_SEC

        self.polling_timeout_ms = timeout_sec * 1000

    def send(self, mode, script):
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:

            # ��³����
            sock.connect((socket.gethostname(), self.port))
            self.logger.info("==sock.connect port: %d" % self.port)

            # �׵����������
            self._send_data(sock, mode, script)
            self.logger.info("==sock._send_data mode: %s, script: %s" % (mode, script))
            sock.shutdown(socket.SHUT_WR)
            self.logger.info("==sock.shutdown")

            # ��������
            return self._recv_data(sock)

    def _send_data(self, sock, mode, script):
        bin = script.encode('utf-8')
        size = len(bin) + ScriptSender.INT32_BYTES

        sock.sendall(size.to_bytes(ScriptSender.INT32_BYTES, 'big'))
        sock.sendall(mode.to_bytes(ScriptSender.INT32_BYTES, 'big'))
        sock.sendall(bin)

    def _recv_data(self, sock):
        size = self._recv_int32(sock)
        body = self._recv_binary(sock, size)
        self.logger.info("==sock._recv_data size: %d, body: %s" % (size, body))
        return json.loads(body.decode('utf-8'))

    def _recv_binary(self, sock, size):
        poll = select.poll()
        poll.register(sock, select.POLLIN)

        data = b''
        while len(data) < size:
            ### SST000-01542 add timeout
            ready = poll.poll(self.polling_timeout_ms)
            if not ready:
                ### SST000-01542 raise timeout error
                raise TimeoutError('Timeout of poll.poll().')

            remine = size - len(data)

            d1 = sock.recv(remine)
            if len(d1) == 0:
                raise Exception('disconnect.')

            data += d1

        return data

    def _recv_int32(self, sock):
        data = self._recv_binary(sock, ScriptSender.INT32_BYTES)
        return int.from_bytes(data, 'big')





－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－--------------------


MDAS の対象ホストが systemd を使う構成なら、常駐 service ではなく Type=oneshot の service と timer を組み合わせるのがよいです。timer が毎日 00:40 に service を起動し、flock で同じ処理の重複起動を防ぎます。

---
# /etc/systemd/system/mdas-daily-job.service
[Unit]
Description=MDAS daily job

[Service]
Type=oneshot
User=ifour
WorkingDirectory=/console/mdas
RuntimeDirectory=mdas-daily-job
ExecStart=/usr/bin/flock -n /run/mdas-daily-job/lock /console/mdas/bin/対象のスクリプト.sh
---
---
# /etc/systemd/system/mdas-daily-job.timer
[Unit]
Description=Run MDAS daily job at 00:40

[Timer]
OnCalendar=*-*-* 00:40:00
AccuracySec=1s
Unit=mdas-daily-job.service

[Install]
WantedBy=timers.target
---
反映・有効化と確認は次のとおりです。
---
sudo systemctl daemon-reload
sudo systemctl start mdas-daily-job.service  # まず手動で動作確認
sudo systemctl enable --now mdas-daily-job.timer
systemctl list-timers mdas-daily-job.timer
journalctl -u mdas-daily-job.service
---
flock -n がロックを取れない場合は、2つ目の起動をせず終了します。その場合は systemd 上で service が失敗扱いになるため、ログで「重複によるスキップ」と実際の処理失敗を区別したい場合は、ロック競合を正常終了に変換するラッパーを用意します。また、スクリプトを service 以外から直接実行する経路があるなら、その経路にも同じロックを適用してください。

このリポジトリでは、現在の cron 登録は 6300ES6a_cron にあり、Installer の Makefile はその設定をインストール成果物へコピーしています。移行時は、対象処理の cron 登録を外して二重スケジュールを避け、service/timer の unit ファイルを同じインストーラー経路で配置・有効化するのがよいです。実際のスクリプトは User=ifour で環境変数や権限が足りるか確認してください。既存の例 executor_ErrorEvent_cron.sh は、cron 起動時の環境設定を独自に読み込んでいます。

なお、timer の Persistent=true を追加すると、停止中に実行時刻を逃した場合、起動後に実行します。再起動後の追いつき実行が必要な場合だけ有効にしてください。
