#!/usr/bin/env python3
"""同じ装置・ファイルグループで動作中のDataGenerator重複を検出する。

MDAS装置上で、/procを読めるifourユーザーまたはrootとして実行する。

使用例:
    python3 check_data_generator_duplicates.py
    python3 check_data_generator_duplicates.py --tool-id TOOL001 --file-group LotSeq
    python3 check_data_generator_duplicates.py --json

終了コード:
    0: 重複なし
    1: 重複あり
    2: プロセス走査が不完全、または/procを参照できない
"""

import argparse
import json
import os
import sys

# 検出対象とする定期実行用DataGenerator Executorのスクリプト名。
EXECUTOR_NAME = "executor_DataGenerator.py"


def _get_executor_args(argv):
    """プロセス引数を解析し、対象外ならNoneを返す。"""
    # コマンドラインが取得できなければ対象プロセスではない。
    if not argv:
        return None

    # 起動形式によりargv[0]がスクリプト名かPython実行ファイル名かを判定する。
    executable = os.path.basename(argv[0])
    # スクリプトを直接実行している形式では、スクリプト名の位置は0。
    if executable == EXECUTOR_NAME:
        script_index = 0
    # python/python3経由なら、次の引数が対象スクリプトかを確認する。
    elif executable.startswith("python") and len(argv) > 1:
        # 別スクリプトを起動したPythonプロセスは対象外にする。
        if os.path.basename(argv[1]) != EXECUTOR_NAME:
            return None
        # Pythonコマンドの次にある引数がスクリプト名。
        script_index = 1
    # 上記以外の実行形式は対象外にする。
    else:
        return None

    # スクリプトより後ろにある引数だけを解析対象にする。
    args = argv[script_index + 1 :]
    # --option以外の位置引数を格納する。
    positional = []
    # --recoverが指定されているかを記録する。
    recovery = False
    # 引数リストを先頭から確認するための位置。
    index = 0
    # 全引数を順番に確認する。
    while index < len(args):
        # 現在調べている引数。
        arg = args[index]
        # Recovery実行かどうかを記録する。
        if arg == "--recover":
            recovery = True
        # 値を別引数で渡す--loglevel形式は、値の分も読み飛ばす。
        elif arg == "--loglevel":
            index += 1
            # --loglevelの値が存在しなければ有効な起動引数ではない。
            if index >= len(args):
                return None
        # --loglevel=INFO形式は値込みのため、追加処理は不要。
        elif arg.startswith("--loglevel="):
            pass
        # 想定していないオプションがあれば誤解析を防ぐため対象外にする。
        elif arg.startswith("-"):
            return None
        # station、開始時刻、終了時刻、ファイルグループを順に記録する。
        else:
            positional.append(arg)
        # 次の引数へ進む。
        index += 1

    # station・開始時刻・終了時刻・ファイルグループの4項目が必要。
    if len(positional) < 4:
        return None

    # 後続処理で使う名前付きデータとして解析結果を返す。
    return {
        "station": positional[0],
        "start_time": positional[1],
        "end_time": positional[2],
        "file_group": positional[3],
        "recovery": recovery,
    }


def _read_process(proc_path):
    """/proc/PIDから対象プロセス情報を読み、(情報, 読み取りエラー)を返す。"""
    try:
        # まず短いプロセス名だけを読み、Python以外を早期に除外する。
        with open(os.path.join(proc_path, "comm"), "r", encoding="ascii") as comm_file:
            process_name = comm_file.read().strip()
    # 読み込み前にプロセスが終了した場合は、通常の競合として無視する。
    except FileNotFoundError:
        return None, None
    # 権限不足などは読み取りエラーとして呼び出し元へ知らせる。
    except OSError as error:
        return None, error

    # DataGeneratorはpython3で起動するため、Python以外のPIDはここで除外する。
    if not process_name.startswith("python"):
        return None, None

    try:
        # cmdlineはNUL区切りなので、引数配列へ戻して読み込む。
        with open(os.path.join(proc_path, "cmdline"), "rb") as cmdline_file:
            # OS依存の文字コード変換を使い、バイト列の引数を文字列にする。
            argv = [os.fsdecode(arg) for arg in cmdline_file.read().split(b"\0") if arg]
    # 読み込み中にプロセスが終了した場合は、通常の競合として無視する。
    except FileNotFoundError:
        return None, None
    # 権限不足など、ほかの読み取り失敗は呼び出し元へ知らせる。
    except OSError as error:
        return None, error

    # 実行コマンドからDataGeneratorの引数を取り出す。
    executor_args = _get_executor_args(argv)
    # DataGenerator以外、または引数が不正なプロセスは無視する。
    if executor_args is None:
        return None, None

    # /procのディレクトリ名がPIDなので数値として記録する。
    pid = int(os.path.basename(proc_path))
    # 親PIDはstatから読み取るため、初期値を未取得にしておく。
    ppid = None
    # 装置識別用の環境変数をここへ格納する。
    environment = {}
    # stat/environの読取エラーがあれば保持する。
    error = None

    try:
        # statにはPID、状態、PPIDなどのカーネル管理情報が入っている。
        with open(os.path.join(proc_path, "stat"), "r", encoding="ascii") as stat_file:
            stat = stat_file.read()
        # comm欄は括弧で囲まれるため、最後の')'より後を項目分解する。
        stat_fields = stat[stat.rfind(")") + 2 :].split()
        # 括弧後の項目は状態が先頭、PPIDが2番目。
        ppid = int(stat_fields[1])
    # 読み込み途中でプロセスが消えた場合は対象から外す。
    except (FileNotFoundError, ProcessLookupError):
        return None, None
    # statが読めない場合も後で判定不完全として扱う。
    except (OSError, IndexError, ValueError) as stat_error:
        error = stat_error

    try:
        # 起動時の環境変数から装置を特定する情報を得る。
        with open(os.path.join(proc_path, "environ"), "rb") as environ_file:
            # environはNUL区切りのKEY=VALUE形式。
            for item in environ_file.read().split(b"\0"):
                # KEY=VALUE形式でない要素は無視する。
                if b"=" in item:
                    # 最初の'='だけで分割し、値に'='があっても壊さない。
                    key, value = item.split(b"=", 1)
                    # 装置名とシリアル番号だけを保持する。
                    if key in (b"TOOL_ID", b"SERIAL_NUM"):
                        environment[os.fsdecode(key)] = os.fsdecode(value)
    # 読み込み前にプロセスが終了した場合は対象から外す。
    except (FileNotFoundError, ProcessLookupError):
        return None, None
    # 権限不足などの環境変数読取エラーは記録する。
    except OSError as environ_error:
        error = error or environ_error

    # TOOL_IDが最も一意な装置識別子なので優先する。
    tool_id = environment.get("TOOL_ID")
    # TOOL_IDが未設定の場合に備えてシリアル番号も取得する。
    serial_num = environment.get("SERIAL_NUM")
    # TOOL_IDがなければSERIAL_NUMで同じ装置のプロセスをまとめる。
    equipment = tool_id or serial_num
    # 装置を識別できないプロセスは誤判定防止のため結果から除外し、エラーにする。
    if not equipment:
        return None, error or RuntimeError("TOOL_ID and SERIAL_NUM are unavailable")

    # 重複判定と人による確認に必要な情報をひとつの辞書へまとめる。
    record = {
        "pid": pid,
        "ppid": ppid,
        "tool_id": tool_id,
        "serial_num": serial_num,
        "equipment": equipment,
        "equipment_source": "TOOL_ID" if tool_id else "SERIAL_NUM",
        "station": executor_args["station"],
        "start_time": executor_args["start_time"],
        "end_time": executor_args["end_time"],
        "file_group": executor_args["file_group"],
        "recovery": executor_args["recovery"],
    }
    # 情報と、読み取り中に発生した可能性のあるエラーを返す。
    return record, error


def scan_processes(proc_root="/proc"):
    """/procを走査し、DataGenerator一覧と読み取りエラーを返す。"""
    # 対象プロセスの情報を追加していくリスト。
    records = []
    # 権限不足など、走査中に見つかった問題を追加するリスト。
    errors = []

    try:
        # /proc直下にはPID名のディレクトリと、その他のシステム情報が並ぶ。
        entries = os.listdir(proc_root)
    # /proc自体がない、または一覧取得できない場合は走査不能として返す。
    except OSError as error:
        return records, [f"Cannot scan {proc_root}: {error}"]

    # /proc直下のエントリーを1つずつ確認する。
    for entry in entries:
        # PIDではないエントリーはプロセス情報ではないので読み飛ばす。
        if not entry.isdigit():
            continue
        # PIDディレクトリからコマンドライン等を読み取る。
        record, error = _read_process(os.path.join(proc_root, entry))
        # 取得に失敗したPIDを記録し、全体が完全な走査でないことを示す。
        if error is not None:
            errors.append(f"PID {entry}: {error}")
        # 対象DataGeneratorと確認できたプロセスだけ一覧に追加する。
        if record is not None:
            records.append(record)

    # 検出結果とエラーの両方を呼び出し元に返す。
    return records, errors


def find_duplicates(records, tool_id=None, file_group=None):
    """装置識別子とファイルグループが同じプロセスをまとめる。"""
    # (装置識別子, ファイルグループ)ごとのプロセス一覧を作る。
    groups = {}
    # スキャンで見つかったDataGeneratorを1件ずつ確認する。
    for record in records:
        # TOOL_ID指定時は、指定装置以外のプロセスを除外する。
        if tool_id and record["tool_id"] != tool_id:
            continue
        # FileGroup指定時は、指定グループ以外のプロセスを除外する。
        if file_group and record["file_group"] != file_group:
            continue
        # 同一装置かつ同一FileGroupを同じグループにする。
        key = (record["equipment"], record["file_group"])
        # 該当する実行プロセスをグループへ追加する。
        groups.setdefault(key, []).append(record)

    # 同じキーのプロセスが2件以上あるグループだけを重複として返す。
    return {key: items for key, items in groups.items() if len(items) > 1}


def _parse_args():
    # --help表示にスクリプトの説明・使用例を使う。
    parser = argparse.ArgumentParser(description=__doc__)
    # 指定された装置だけを調べるための任意フィルター。
    parser.add_argument("--tool-id", help="Only inspect this TOOL_ID")
    # 指定されたFileGroupだけを調べるための任意フィルター。
    parser.add_argument("--file-group", help="Only inspect this file group")
    # 集計システムなどから扱いやすいJSON出力を有効にする。
    parser.add_argument(
        "--json", action="store_true", help="Print machine-readable JSON"
    )
    # テストで疑似/procを渡せる内部用引数。通常のヘルプには表示しない。
    parser.add_argument("--proc-root", default="/proc", help=argparse.SUPPRESS)
    # コマンドラインを解析して呼び出し元へ返す。
    return parser.parse_args()


def main():
    # コマンドラインの絞り込み条件と出力形式を取得する。
    options = _parse_args()
    # 現在動作中のDataGeneratorプロセスを一覧化する。
    records, errors = scan_processes(options.proc_root)
    # 装置とFileGroupでまとめ、2件以上のグループを抽出する。
    duplicates = find_duplicates(records, options.tool_id, options.file_group)

    # JSON出力に使う結果全体を組み立てる。
    result = {
        # 重複グループが1つでもあるかを真偽値で返す。
        "duplicates_found": bool(duplicates),
        # 読み取りエラーがなければ全プロセスを調べられたとする。
        "scan_complete": not errors,
        # 対象として解析できたDataGeneratorの総数。
        "process_count": len(records),
        # JSONでも扱いやすいよう重複グループを辞書の配列に変換する。
        "duplicates": [
            {"equipment": key[0], "file_group": key[1], "processes": processes}
            for key, processes in sorted(duplicates.items())
        ],
        "scan_errors": errors,
    }

    # 機械処理向けにJSONを整形して出力する。
    if options.json:
        print(json.dumps(result, ensure_ascii=False, indent=2))
    # 人が端末で確認する場合は、重複グループごとに詳細を表示する。
    else:
        # 重複があれば、装置・FileGroup・件数を表示する。
        if duplicates:
            for (equipment, file_group), processes in sorted(duplicates.items()):
                print(
                    f"DUPLICATE equipment={equipment} file_group={file_group} "
                    f"count={len(processes)}"
                )
                for process in sorted(processes, key=lambda item: item["pid"]):
                    print(
                        "  PID={pid} PPID={ppid} station={station} "
                        "range={start_time}-{end_time} recovery={recovery}".format(
                            **process
                        )
                    )
        # 重複がなければ、調べたプロセス数とともに正常結果を表示する。
        else:
            print(
                f"No duplicate DataGenerator processes found ({len(records)} scanned)."
            )

        # 読めなかったプロセス情報を標準エラーへ明示する。
        for error in errors:
            print(f"SCAN ERROR: {error}", file=sys.stderr)

    # 重複ありをスクリプト終了コード1で呼び出し元へ知らせる。
    if duplicates:
        return 1
    # 重複なしでも走査に失敗があれば、判定不完全として終了コード2を返す。
    if errors:
        return 2
    # 重複がなく、走査も完了した場合の正常終了コード。
    return 0


# このファイルを直接起動した場合だけmainを実行する。
if __name__ == "__main__":
    # mainの戻り値をプロセス終了コードとしてOSへ返す。
    sys.exit(main())
