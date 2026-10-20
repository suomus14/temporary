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
