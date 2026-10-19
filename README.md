#!/usr/bin/env python3
"""同じ装置・ファイルグループで動作中のDataGenerator重複を検出する。"""

import os  # パス名を処理する機能を読み込む。
import sys  # 終了コードを返す機能を読み込む。

# 検出対象のExecutor名を設定する。
EXECUTOR_NAME = "executor_DataGenerator.py"


# コマンドラインからDataGeneratorの実行引数を取得する。
def get_exec_args(argv):
    # 対象Executor以外のプロセスを除外する。
    if len(argv) < 2 or os.path.basename(argv[1]) != EXECUTOR_NAME:
        # 対象外を示す値を返す。
        return None

    # Executor名より後ろの引数を取得する。
    args = argv[2:]
    # 先頭引数がRecovery指定か判定する。
    recovery = bool(args and args[0] == "--recover")
    # Recovery指定があれば引数から取り除く。
    if recovery:
        args = args[1:]
    # loglevel引数がない形式を対象外にする。
    if not args or not args[0].startswith("--loglevel="):
        # 不正な引数を示す値を返す。
        return None

    # loglevel引数より後ろの引数を取得する。
    args = args[1:]
    # 必須の位置引数がそろっているか判定する。
    if len(args) < 4:
        # 必須引数が不足した場合は対象外を返す。
        return None

    # 後続処理で使う引数を名前付きデータにまとめて返す。
    return {
        "station": args[0],  # station名を設定する。
        "start_time": args[1],  # 開始時刻を設定する。
        "end_time": args[2],  # 終了時刻を設定する。
        "file_group": args[3],  # ファイルグループ名を設定する。
        "recovery": recovery,  # Recovery実行かどうかを設定する。
    }


# /procのプロセス情報からDataGeneratorの情報を取得する。
def read_proc(proc_path):
    # ディレクトリ名からPIDを取得する。
    pid = int(os.path.basename(proc_path))

    try:
        # プロセス名を読み取る。
        with open(os.path.join(proc_path, "comm"), "r", encoding="ascii") as comm_file:
            proc_name = comm_file.read().strip()
        # Python以外のプロセスを除外する。
        if not proc_name.startswith("python"):
            # 対象外を示す値を返す。
            return None

        # NUL区切りのコマンドラインを読み取る。
        with open(os.path.join(proc_path, "cmdline"), "rb") as cmdline_file:
            # コマンドラインを引数の配列に変換する。
            argv = [os.fsdecode(arg) for arg in cmdline_file.read().split(b"\0") if arg]
        # コマンドラインからExecutorの引数を取得する。
        exec_args = get_exec_args(argv)
        # 対象Executor以外のプロセスを除外する。
        if exec_args is None:
            # 対象外を示す値を返す。
            return None

        # 装置識別用の環境変数を格納する。
        environment = {}
        # プロセスの環境変数を読み取る。
        with open(os.path.join(proc_path, "environ"), "rb") as environ_file:
            # NUL区切りの環境変数を順に確認する。
            for item in environ_file.read().split(b"\0"):
                # KEY=VALUE形式の項目だけを処理する。
                if b"=" in item:
                    # 最初の等号でキーと値に分割する。
                    key, value = item.split(b"=", 1)
                    # 装置識別に使う環境変数だけを保存する。
                    if key in (b"TOOL_ID", b"SERIAL_NUM"):
                        # キーと値を文字列に変換して保存する。
                        environment[os.fsdecode(key)] = os.fsdecode(value)
    except (FileNotFoundError, ProcessLookupError):
        # 読み取り前に終了したプロセスを除外する。
        # 対象外を示す値を返す。
        return None

    # TOOL_IDを取得する。
    tool_id = environment.get("TOOL_ID")
    # シリアル番号を取得する。
    serial_num = environment.get("SERIAL_NUM")
    # TOOL_IDを優先して装置識別子を決定する。
    equipment = tool_id or serial_num
    # 装置を識別できないプロセスをエラーにする。
    if not equipment:
        # 識別情報がない理由を呼び出し元へ伝える。
        raise ValueError("TOOL_ID and SERIAL_NUM are unavailable")

    # 重複判定に使うプロセス情報をまとめて返す。
    return {
        "pid": pid,  # プロセスIDを設定する。
        "tool_id": tool_id,  # TOOL_IDを設定する。
        "serial_num": serial_num,  # シリアル番号を設定する。
        "equipment": equipment,  # 装置識別子を設定する。
        "equipment_source": "TOOL_ID"
        if tool_id
        else "SERIAL_NUM",  # 識別元を設定する。
        "station": exec_args["station"],  # station名を設定する。
        "start_time": exec_args["start_time"],  # 開始時刻を設定する。
        "end_time": exec_args["end_time"],  # 終了時刻を設定する。
        "file_group": exec_args["file_group"],  # ファイルグループ名を設定する。
        "recovery": exec_args["recovery"],  # Recovery実行かどうかを設定する。
    }


# /procを走査してDataGeneratorの一覧を取得する。
def scan_proc(proc_root="/proc"):
    # 検出したプロセス情報を格納する。
    records = []
    # 走査が最後まで完了したかを記録する。
    complete = True

    # /proc直下のエントリを取得する。
    try:
        entries = os.listdir(proc_root)
    except OSError as error:
        # /procを読み取れない場合にエラーを表示する。
        print(f"ERROR: cannot read {proc_root}: {error}")
        # 空の結果と走査失敗を返す。
        return records, False

    # /proc直下のエントリを順に確認する。
    for entry in entries:
        # PID名ではないエントリを無視する。
        if not entry.isdigit():
            # 次のエントリへ進む。
            continue

        try:
            # PIDディレクトリから対象プロセス情報を取得する。
            record = read_proc(os.path.join(proc_root, entry))
        except (OSError, ValueError) as error:
            # プロセス情報を読み取れない場合にエラーを表示する。
            # PIDとエラー内容を標準出力へ表示する。
            print(f"ERROR PID {entry}: {error}")
            # 走査が不完全であることを記録する。
            complete = False
            # 次のPIDの確認へ進む。
            continue
        # 対象プロセスだけを一覧に追加する。
        if record is not None:
            # 取得したプロセス情報を保存する。
            records.append(record)

    # プロセス一覧と走査完了状態を返す。
    return records, complete


# 装置とファイルグループが重複するプロセスを抽出する。
def find_duplicates(records):
    # 装置とファイルグループごとのプロセス一覧を格納する。
    groups = {}
    # 検出したプロセスを順に分類する。
    for record in records:
        # 同じ装置かつ同じファイルグループを同じキーにする。
        key = (record["equipment"], record["file_group"])
        # 該当するグループへプロセスを追加する。
        groups.setdefault(key, []).append(record)

    # 2件以上のプロセスがあるグループだけを返す。
    return {key: items for key, items in groups.items() if len(items) > 1}


# 重複結果を表示して終了コードを返す。
def main():
    # DataGeneratorのプロセス一覧と走査状態を取得する。
    records, complete = scan_proc()
    # 装置とファイルグループが重複するプロセスを抽出する。
    duplicates = find_duplicates(records)

    # 重複したグループを順に表示する。
    for (equipment, file_group), processes in sorted(duplicates.items()):
        # 装置、ファイルグループ、重複数を表示する。
        print(
            f"DUPLICATE equipment={equipment} file_group={file_group} "
            f"count={len(processes)}"
        )
        # グループ内のプロセスをPID順に表示する。
        for process in sorted(processes, key=lambda item: item["pid"]):
            # プロセスの実行条件を表示する。
            print(
                "  PID={pid} station={station} "
                "range={start_time}-{end_time} recovery={recovery}".format(**process)
            )

    # 重複がない場合に走査件数を表示する。
    if not duplicates:
        print(f"No duplicate DataGenerator processes found ({len(records)} scanned).")
    # 重複がある場合に重複ありの終了コードを返す。
    if duplicates:
        return 1
    # 走査が不完全な場合にエラー終了コードを返す。
    if not complete:
        return 2
    # 重複がなく走査も完了した場合に正常終了コードを返す。
    return 0


# 直接実行された場合にmainを実行する。
if __name__ == "__main__":
    # mainの終了コードでプロセスを終了する。
    sys.exit(main())
