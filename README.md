----------------------------------------------------------------------------------------------------
Client.cpp
----------------------------------------------------------------------------------------------------
/**
 * @file    Client.cpp
 * @author  Copyright(C) Canon Inc. All right reserved.
 * @date    2014/01/10
 * @brief   クライアントクラス
 * @par     説明
 *  - TCP通信のクライアント機能を実現するクラス
 */
#include <cstring>
#include "I_ClientTcpIo.hpp"
#include "Client.hpp"
#include "TcpUtil.hpp"

/**
 * @brief        コンストラクタ
 */
Client::Client(void)
{
	client_.local.addr_len = sizeof(client_.local.addr);
	client_.remote.addr_len = sizeof(client_.remote.addr);
}

Client::Client(in_addr hostInfo, int portInfo)
{
	client_.remote.addr.sin_family = AF_INET;
	client_.remote.addr.sin_port = portInfo;
	client_.remote.addr.sin_addr = hostInfo;
	client_.local.addr_len = sizeof(client_.local.addr);
	client_.remote.addr_len = sizeof(client_.remote.addr);
}

/**
 * @brief        デストラクタ
 */
Client::~Client()
{
}

/**
 * @brief        初期化処理
 * @param[in]    host        ホスト名
 * @param[in]    service     サービス名
 * @param[out]   hostInfo    IPアドレス情報
 * @param[out]   portInfo    ポート番号情報
 *
 * @retval       なし
 */
void Client::TcpInit(const char *host, const char *service, in_addr *hostInfo, int *portInfo)
{
	log_printf(LV_DEBUG, "start\n");

	try
	{
		// ホスト名からIPアドレスを取得
		if (hostInfo->s_addr == 0)
		{
			tcpGetHostByName(host, hostInfo);
			log_printf(LV_DEBUG, "hostname = %s, hostInfo = %d\n", host, hostInfo->s_addr);
		}

		// サービス名からポート番号を取得
		if (*portInfo == 0)
		{
			tcpGetServiceByName(service, portInfo);
			log_printf(LV_DEBUG, "service = %s, portInfo = %d\n", service, *portInfo);
		}

		client_.remote.addr.sin_family = AF_INET;
		client_.remote.addr.sin_port = *portInfo;
		client_.remote.addr.sin_addr.s_addr = hostInfo->s_addr;

		log_printf(LV_DEBUG, "hostInfo = %d\n", client_.remote.addr.sin_addr.s_addr);
		log_printf(LV_DEBUG, "portInfo = %d\n", client_.remote.addr.sin_port);
		log_printf(LV_DEBUG, "len      = %d\n", client_.remote.addr_len);
	}
	catch (int error)
	{
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		throw error;
	}
	catch (...)
	{
		log_printf(LV_ERROR, "error.\n");
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        サーバ接続
 * @param[inout] sd          ソケットディスクリプタ
 * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
 *
 * @retval       なし
 */
void Client::TcpConnect(int *sd, int connect)
{
	int err = 0;   ///< エラー戻り値
	int sd_ = *sd; ///< ソケットディスクリプタ

	log_printf(LV_DEBUG, "start\n");

	// 常時接続型は初回時のみソケットを生成する
	if ((connect == TCP_CONNECT) && (sd_ >= 0))
	{
		log_printf(LV_DEBUG, "sd(client:after socket) = %d\n", sd_);
		log_printf(LV_DEBUG, "end(no exec.)\n");
		return;
	}

	try
	{
		// ソケットを生成
		sd_ = socket(AF_INET, SOCK_STREAM, 0);
		if (sd_ < 0)
		{
			log_printf(LV_ERROR, "socket error.(sd = %d, errno = %d)\n", sd_, errno);
			throw ERR_CLIENT_SYSTEM;
		}
		log_printf(LV_DEBUG, "sd(client:after socket) = %d\n", sd_);

		// TIME_WAIT状態を回避する
		struct linger optval;
		optval.l_onoff = 1;
		optval.l_linger = 0;
		err = setsockopt(sd_, SOL_SOCKET, SO_LINGER, (void *)&optval, sizeof(optval));
		if (err < 0)
		{
			throw ERR_CLIENT_SYSTEM;
		}

		//SST000-01635:Set tcp_nodelay for changing Nagle mode OFF(for avoid 40msec tcp delay) 
		tcpConnect(sd_);

		// サーバ接続
		err = getsockname(sd_, (struct sockaddr *)&client_.local.addr, (SOCKLEN *)&client_.local.addr_len);
		if (err < 0)
		{
			log_printf(LV_ERROR, "getsockname error.(err = %d, errno = %d)\n", err, errno);
			throw ERR_CLIENT_GETSOCKNAME;
		}
		log_printf(LV_DEBUG, "local.sin_port = %d\n", client_.local.addr.sin_port);
		log_printf(LV_DEBUG, "local.sin_addr = %d\n", client_.local.addr.sin_addr.s_addr);

		// ローカルアドレスを取得
		*sd = sd_;
	}
	catch (int error)
	{
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		close(*sd); // ソケットディスクリプタを取得
		throw error;
	}
	catch (...)
	{
		log_printf(LV_ERROR, "error.\n");
		close(*sd); // ソケットディスクリプタを閉じておく
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        サーバからのデータ読み込み
 *               読み込みは1メッセージ
 * @param[in]    sd          ソケットディスクリプタ
 * @param[out]   size        返信データサイズ
 * @param[out]   data        返信データ
 * @param[out]   error       エラー情報
 * @param[out]   header      ヘッダ(固定文字列)
 * @param[out]   funcId      関数ID
 * @param[out]   parity1     パリティ情報1
 * @param[out]   parity2     パリティ情報2
 *
 * @retval       なし
 */
void Client::TcpRead(int sd, int *size, char **data, int *error, char *header, int *funcId, int *parity1, int *parity2)
{
	log_printf(LV_DEBUG, "start\n");

	// 中間メッセージ、返信メッセージ共にヘッダは8バイトである。
	// 最初に8バイトデータを読み込みヘッダ情報を取得する。
	int headRecvSize = HEADER_SIZE;
	char headData[HEADER_SIZE + 1];
	memset(headData, 0, sizeof(headData));
	tcpSelectRfs(sd, TIMEOUT);
	tcpReadBytes(sd, headRecvSize, headData);

	if (strncmp(headData, "InfmHead", sizeof(headData)) == 0)
	{

		// 中間メッセージを受信した場合、残りのデータを読み込む
		// 関数ID(4)＋パリティ情報(8)＋エラー情報(4)＋中間フッタ(8)
		S_TcpMsgC msgC;
		memset(&msgC, 0, sizeof(msgC));
		int restRecvSize = sizeof(msgC) - sizeof(msgC.header);
		char *restData = new char[restRecvSize];
		if (restData == NULL)
		{
			throw ERR_CLIENT_MALLOC;
		}
		memset(restData, 0, restRecvSize * sizeof(char));
		tcpReadBytes(sd, restRecvSize, restData);
		memcpy(&msgC.header, headData, headRecvSize);
		memcpy(&msgC.funcId, restData, restRecvSize);
		if (restData != NULL)
		{
			delete[] restData;
			restData = NULL;
		}
		memcpy(header, headData, headRecvSize);
		*funcId = ntohl(msgC.funcId);
		*parity1 = ntohl(msgC.parity1);
		*parity2 = ntohl(msgC.parity2);
		*error = ntohl(msgC.error);
	}
	else if (strncmp(headData, "RecvHead", sizeof(headData)) == 0)
	{

		// 返信メッセージを受信した場合
		// 関数ID(4)＋パリティ情報(8)＋データサイズ(4)＋エラー情報(4)＋返信データ(可変)＋返信フッタ(8)
		// 1. エラー情報(4)までのデータを読み込む
		S_TcpMsgE msgE;
		memset(&msgE, 0, sizeof(msgE));
		int fixRecvSize = sizeof(msgE.funcId) + sizeof(msgE.parity1) + sizeof(msgE.parity2) + sizeof(msgE.size) + sizeof(msgE.error);
		char *fixData = new char[fixRecvSize];
		if (fixData == NULL)
		{
			throw ERR_CLIENT_MALLOC;
		}
		memset(fixData, 0, fixRecvSize * sizeof(char));
		tcpReadBytes(sd, fixRecvSize, fixData);
		memcpy(&msgE.header, headData, headRecvSize);
		memcpy(&msgE.funcId, fixData, fixRecvSize);
		if (fixData != NULL)
		{
			delete[] fixData;
			fixData = NULL;
		}

		// 2. 取得したデータサイズの情報を元に残りのデータを読み込む
		msgE.size = ntohl(msgE.size);
		if (msgE.size <= 0)
		{
			log_printf(LV_DEBUG, "parameter size is illegal.(%d)\n", msgE.size);
			throw ERR_CLIENT_RECEIVE;
		}
		*size = msgE.size;
		int restRecvSize = *size + sizeof(msgE.footer);
		if (*data != NULL)
		{
			delete[] *data;
			*data = NULL;
		}
		*data = new char[*size];
		char *restData = new char[restRecvSize];
		if ((*data == NULL) || (restData == NULL))
		{
			throw ERR_CLIENT_MALLOC;
		}
		memset(*data, 0, *size * sizeof(char));
		memset(restData, 0, restRecvSize * sizeof(char));
		tcpReadBytes(sd, restRecvSize, restData);
		memcpy(*data, &restData[0], *size);
		memcpy(&msgE.footer, &restData[*size], sizeof(msgE.footer));
		if (restData != NULL)
		{
			// dataの領域解放は通常上位で行う
			delete[] restData;
			restData = NULL;
		}
		memcpy(header, headData, headRecvSize);
		*funcId = ntohl(msgE.funcId);
		*parity1 = ntohl(msgE.parity1);
		*parity2 = ntohl(msgE.parity2);
		*error = ntohl(msgE.error);
	}
	else
	{
		// ヘッダが見つからない
		log_printf(LV_ERROR, "not find header error.(%s)\n", headData);
		throw ERR_CLIENT_RECEIVE;
	}

	log_printf(LV_DEBUG, "header  = %s\n", header);
	log_printf(LV_DEBUG, "funcId  = %d\n", *funcId);
	log_printf(LV_DEBUG, "parity1 = %d\n", *parity1);
	log_printf(LV_DEBUG, "parity2 = %d\n", *parity2);
	log_printf(LV_DEBUG, "error   = %d\n", *error);

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        サーバにデータ書き込み
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    size        送信データサイズ
 * @param[in]    data        送信データ
 *
 * @retval       なし
 */
void Client::TcpWrite(int sd, int size, char *data)
{
	log_printf(LV_DEBUG, "start\n");

	// サーバにデータ書き込み
	tcpSelectWfs(sd, TIMEOUT);
	tcpWriteBytes(sd, size, data);

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        サーバ切断
 * @param[inout] sd          ソケットディスクリプタ
 * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
 *
 * @retval       なし
 */
void Client::TcpClose(int *sd, int connect)
{
	int err = 0;   ///< エラー戻り値
	int sd_ = *sd; ///< ソケットディスクリプタ

	log_printf(LV_DEBUG, "start\n");

	// 常時接続型は切断しない
	if (connect == TCP_CONNECT)
	{
		log_printf(LV_DEBUG, "end(no exec.)\n");
		return;
	}

	// 書き込みを閉じ、相手にFINを送る
	// 相手からのFINを待たずに強制終了する
	err = shutdown(sd_, SHUT_RDWR);
	if (err < 0)
	{
		log_printf(LV_ERROR, "shutdown error.(err = %d, errno = %d)\n", err, errno);
	}

	// ソケットを閉じる
	err = close(sd_);
	if (err < 0)
	{
		log_printf(LV_ERROR, "close error.(err = %d, errno = %d)\n", err, errno);
	}
	log_printf(LV_DEBUG, "sd(client:after close) = %d\n", *sd);
	*sd = -1;

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        ホスト名からIPアドレスを取得
 * @param[in]    host        ホスト名
 * @param[out]   in_addr     IPアドレス(struct in_addr)
 *
 * @retval       なし
 */
void Client::tcpGetHostByName(const char *host, struct in_addr *in_addr)
{
	struct hostent *h_info; ///< hostent構造体

	// ホスト名からIPアドレスを取得
	h_info = gethostbyname(host);
	if (h_info == NULL)
	{
		log_printf(LV_ERROR, "gethostbyname error.(%d)\n", errno);
		throw ERR_CLIENT_GETHOSTBYNAME;
	}
	memcpy(&in_addr->s_addr, h_info->h_addr_list[0], h_info->h_length);
	log_printf(LV_DEBUG, "h_length = %d\n", h_info->h_length);
	log_printf(LV_DEBUG, "s_addr = %d\n", (int)in_addr->s_addr);

	return;
}

/**
 * @brief        サービス名からポート番号を取得
 * @param[in]    service     サービス名
 * @param[out]   port        ポート番号
 *
 * @retval       なし
 */
void Client::tcpGetServiceByName(const char *service, int *port)
{
	struct servent *se = NULL; ///< servent構造体

	// サービス名からポート番号を取得
	// プロトコルはTCP固定
	se = getservbyname(service, "tcp");
	if (se == NULL)
	{
		log_printf(LV_ERROR, "getservbyname error.(%d)\n", errno);
		throw ERR_CLIENT_GETSERVBYNAME;
	}
	*port = se->s_port;
	log_printf(LV_DEBUG, "service: %s, port:%d\n", service, *port);

	return;
}

/**
 * @brief        connect実行
 * @param[in]    sd          ソケットディスクリプタ
 *
 * @retval       なし
 */
void Client::tcpConnect(int sd)
{
	int err = 0;		   ///< エラー戻り値
	int connect_errno = 0; ///< 接続時のエラー情報

	log_printf(LV_DEBUG, "start\n");

	try
	{
		// ノンブロッキングモードに変更
		tcpSetNonBlockingMode(sd);

		// サーバ接続
		err = connect(sd, (struct sockaddr *)&(client_.remote.addr), client_.remote.addr_len);
		connect_errno = errno;
		if (err == 0)
		{
			// サーバとクライアントが同一ホストの場合はこのタイミングでコネクションが確立するはず
			// ブロッキングモードに戻す
			tcpSetBlockingMode(sd);
			log_printf(LV_DEBUG, "end\n");
			return;
		}
		if (connect_errno != EINPROGRESS)
		{
			// サーバとクライアントが別ホストの場合は、コネクション確立を開始していればEINPROGRESSが返ってくるはず
			log_printf(LV_ERROR, "connect error = %d\n", connect_errno);
			throw ERR_CLIENT_CONNECT;
		}

		// select実行
		tcpSelect(sd, CONNECT_TIMEOUT);

		// ブロッキングモードに戻す
		tcpSetBlockingMode(sd);
	}
	catch (int error)
	{
		// sdが取れた場合のみブロッキングモードに戻す
		if (error != ERR_CLIENT_19)
		{
			tcpSetBlockingMode(sd);
		}
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		throw error;
	}
	catch (...)
	{
		tcpSetBlockingMode(sd);
		log_printf(LV_ERROR, "error.\n");
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        select実行(読み込み可能待ち)
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    timeout     タイムアウト時間 [sec]
 *
 * @retval       なし
 */
void Client::tcpSelectRfs(int sd, int timeout)
{
	int err = 0;	   ///< エラー戻り値
	fd_set rfs;		   ///< ファイルディスクリプタ
	struct timeval tv; ///< タイムアウト時間設定

	log_printf(LV_DEBUG, "start\n");

	try
	{
		FD_ZERO(&rfs);
		FD_SET(sd, &rfs);

		// セレクトのタイムアウト時間を設定
		tv.tv_sec = timeout;
		tv.tv_usec = 0;

		// select()実行
		if (timeout == -1)
		{
			err = select(sd + 1, &rfs, NULL, NULL, NULL); // タイムアウトなし
		}
		else
		{
			err = select(sd + 1, &rfs, NULL, NULL, &tv);
		}
		if (err == 0)
		{
			// タイムアウト
			log_printf(LV_DEBUG, "select timeout.(%d)\n", (int)tv.tv_sec);
			throw ERR_CLIENT_SELECT_TIMEOUT;
		}
		else if (err < 0)
		{
			log_printf(LV_ERROR, "select error.(err = %d, errno = %d)\n", err, errno);
			throw ERR_CLIENT_SELECT;
		}

		// ファイルディスクリプタにデータあり
		if (FD_ISSET(sd, &rfs))
		{
			int error_code;
			int len = sizeof(error_code);
			;

			// ソケット中の保留のエラーを取得
			err = getsockopt(sd, SOL_SOCKET, SO_ERROR, &error_code, (SOCKLEN *)&len);
			if (err < 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(err = %d, errno = %d)\n", err, errno);
				throw ERR_CLIENT_GETSOCKOPT;
			}

			// 念のためエラーチェック
			if (error_code != 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(%d)\n", error_code);
				throw ERR_CLIENT_GETSOCKOPT_2;
			}
		}
		else
		{
			log_printf(LV_DEBUG, "socket not set.\n");
			throw ERR_CLIENT_SELECT_NO_SOCKET;
		}
	}
	catch (int error)
	{
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		throw error;
	}
	catch (...)
	{
		log_printf(LV_ERROR, "error.\n");
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        select実行(書き込み可能待ち)
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    timeout     タイムアウト時間 [sec]
 *
 * @retval       なし
 */
void Client::tcpSelectWfs(int sd, int timeout)
{
	int err = 0;	   ///< エラー戻り値
	fd_set wfs;		   ///< ファイルディスクリプタ
	struct timeval tv; ///< タイムアウト時間設定

	log_printf(LV_DEBUG, "start\n");

	try
	{
		FD_ZERO(&wfs);
		FD_SET(sd, &wfs);

		// セレクトのタイムアウト時間を設定
		tv.tv_sec = timeout;
		tv.tv_usec = 0;

		// select()実行
		if (timeout == -1)
		{
			err = select(sd + 1, NULL, &wfs, NULL, NULL); // タイムアウトなし
		}
		else
		{
			err = select(sd + 1, NULL, &wfs, NULL, &tv);
		}
		if (err == 0)
		{
			// タイムアウト
			log_printf(LV_DEBUG, "select timeout.(%d)\n", (int)tv.tv_sec);
			throw ERR_CLIENT_SELECT_TIMEOUT;
		}
		else if (err < 0)
		{
			log_printf(LV_ERROR, "select error.(err = %d, errno = %d)\n", err, errno);
			throw ERR_CLIENT_SELECT;
		}

		// ファイルディスクリプタにデータあり
		if (FD_ISSET(sd, &wfs))
		{
			int error_code;
			int len = sizeof(error_code);
			;

			// ソケット中の保留のエラーを取得
			err = getsockopt(sd, SOL_SOCKET, SO_ERROR, &error_code, (SOCKLEN *)&len);
			if (err < 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(err = %d, errno = %d)\n", err, errno);
				throw ERR_CLIENT_GETSOCKOPT;
			}

			// 念のためエラーチェック
			if (error_code != 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(%d)\n", error_code);
				throw ERR_CLIENT_GETSOCKOPT_2;
			}
		}
		else
		{
			log_printf(LV_DEBUG, "socket not set.\n");
			throw ERR_CLIENT_SELECT_NO_SOCKET;
		}
	}
	catch (int error)
	{
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		throw error;
	}
	catch (...)
	{
		log_printf(LV_ERROR, "error.\n");
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        select実行(読み書き可能待ち)
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    timeout     タイムアウト時間 [sec]
 *
 * @retval       なし
 */
void Client::tcpSelect(int sd, int timeout)
{
	int err = 0;	   ///< エラー戻り値
	fd_set rfs, wfs;   ///< ファイルディスクリプタ
	struct timeval tv; ///< タイムアウト時間設定

	log_printf(LV_DEBUG, "start\n");

	try
	{
		// 読み書き両方をセレクトで待つ設定
		FD_ZERO(&rfs);
		FD_SET(sd, &rfs);
		FD_ZERO(&wfs);
		FD_SET(sd, &wfs);

		// セレクトのタイムアウト時間を設定
		tv.tv_sec = timeout;
		tv.tv_usec = 0;

		// select()実行
		if (timeout == -1)
		{
			err = select(sd + 1, &rfs, &wfs, NULL, NULL); // タイムアウトなし
		}
		else
		{
			err = select(sd + 1, &rfs, &wfs, NULL, &tv);
		}
		if (err == 0)
		{
			// タイムアウト
			log_printf(LV_DEBUG, "select timeout.(%d)\n", (int)tv.tv_sec);
			throw ERR_CLIENT_SELECT_TIMEOUT;
		}
		else if (err < 0)
		{
			log_printf(LV_ERROR, "select error.(err = %d, errno = %d)\n", err, errno);
			throw ERR_CLIENT_SELECT;
		}

		// 読み書きができれば接続完了
		if (FD_ISSET(sd, &rfs) || FD_ISSET(sd, &wfs))
		{
			int error_code;
			int len = sizeof(error_code);
			;

			// ソケット中の保留のエラーを取得
			err = getsockopt(sd, SOL_SOCKET, SO_ERROR, &error_code, (SOCKLEN *)&len);
			if (err < 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(err = %d, errno = %d)\n", err, errno);
				throw ERR_CLIENT_GETSOCKOPT;
			}

			// 念のためエラーチェック
			if (error_code != 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(%d)\n", error_code);
				throw ERR_CLIENT_GETSOCKOPT_2;
			}
		}
		else
		{
			log_printf(LV_DEBUG, "socket not set.\n");
			throw ERR_CLIENT_SELECT_NO_SOCKET;
		}
	}
	catch (int error)
	{
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		throw error;
	}
	catch (...)
	{
		log_printf(LV_ERROR, "error.\n");
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        ノンブロッキングモードに変更
 * @param[in]    sd          ソケットディスクリプタ
 *
 * @retval       なし
 */
void Client::tcpSetNonBlockingMode(int sd)
{
	int flags; ///< fcntl戻り値

	flags = fcntl(sd, F_GETFL, 0);
	if (flags < 0)
	{
		log_printf(LV_ERROR, "fcntl error.(err = %d, errno = %d)\n", flags, errno);
		throw ERR_CLIENT_FCNTL;
	}

	if (fcntl(sd, F_SETFL, flags | O_NONBLOCK))
	{
		log_printf(LV_ERROR, "fcntl error.(errno = %d)\n", errno);
		throw ERR_CLIENT_FCNTL;
	}

	return;
}

/**
 * @brief        ブロッキングモードに変更
 * @param[in]    sd          ソケットディスクリプタ
 *
 * @retval       なし
 */
void Client::tcpSetBlockingMode(int sd)
{
	int flags; ///< fcntl戻り値

	flags = fcntl(sd, F_GETFL, 0);
	if (flags < 0)
	{
		log_printf(LV_ERROR, "fcntl error.(err = %d, errno = %d)\n", flags, errno);
		throw ERR_CLIENT_FCNTL;
	}

	if (fcntl(sd, F_SETFL, flags & ~O_NONBLOCK))
	{
		log_printf(LV_ERROR, "fcntl error.(errno = %d)\n", errno);
		throw ERR_CLIENT_FCNTL;
	}

	return;
}

/**
 * @brief        指定したバイトだけデータを受信
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    size        データサイズ
 * @param[out]   data        受信データ
 *
 * @retval       なし
 */
void Client::tcpReadBytes(int sd, int size, char *data)
{
	int restSize = 0; ///< 残受信データサイズ
	int nRecv = 0;	  ///< データ受信結果

	log_printf(LV_DEBUG, "start\n");

	// 指定サイズ受信できるまで繰り返し受信を行う
	restSize = size;
	while (restSize > 0)
	{

		log_printf(LV_DEBUG, "recv size(%d)\n", restSize);
		nRecv = recv(sd, data, restSize, 0);
		if (nRecv < 0)
		{
			// 受信エラー
			log_printf(LV_ERROR, "recv error.(err = %d, errno = %d)\n", nRecv, errno);
			throw ERR_CLIENT_RECV;
		}
		else if (nRecv == 0)
		{
			// 回線切断
			log_printf(LV_ERROR, "connection closed error.\n");
			throw ERR_CLIENT_RECV_DISCONNECT;
		}
		data += nRecv;
		restSize -= nRecv;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        指定したバイトだけデータを送信
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    size        データサイズ
 * @param[in]    data        送信データ
 *
 * @retval       なし
 */
void Client::tcpWriteBytes(int sd, int size, char *data)
{
	int restSize = 0; ///< 残送信データサイズ
	int nSend = 0;	  ///< データ送信結果

	log_printf(LV_DEBUG, "start\n");

	restSize = size;
	while (restSize > 0)
	{

		// SIGPIPEの発生を抑止
		log_printf(LV_DEBUG, "send size(%d)\n", restSize);
		nSend = send(sd, data, restSize, 0);
		if (nSend < 0)
		{
			// 送信エラー
			log_printf(LV_ERROR, "send error.(err = %d, errno = %d)\n", nSend, errno);
			throw ERR_CLIENT_SEND;
		}
		else if (nSend == 0)
		{
			// 回線切断
			log_printf(LV_ERROR, "connection closed error.\n");
			throw ERR_CLIENT_SEND_DISCONNECT;
		}
		data += nSend;
		restSize -= nSend;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}


----------------------------------------------------------------------------------------------------
/**
 * @file    Client.cpp
 * @author  Copyright(C) Canon Inc. All right reserved.
 * @date    2014/01/10
 * @brief   クライアントクラス
 * @par     説明
 *  - TCP通信のクライアント機能を実現するクラス
 */
#include <cstring>
#include "I_ClientTcpIo.hpp"
#include "Client.hpp"
#include "TcpUtil.hpp"

// 目的: 受信・送信が複数回に分割されても、処理全体のタイムアウトが延びないようにする。
// 処理: 指定秒数を現在時刻に加え、全ての部分送受信で共通して使う絶対期限を返す。
// timeout == -1 の場合は期限を設けず、無制限待ちを表す。
static struct timeval tcpCreateDeadline(int timeout)
{
	// 呼び出し元へ返す絶対期限を格納する。
	struct timeval deadline;

	// -1以外の負数は未定義値なので、誤った待ち時間として拒否する。
	if (timeout < -1)
	{
		// 不正な設定値をログに残し、システムエラーとして呼び出し元へ返す。
		log_printf(LV_ERROR, "invalid timeout.(%d)\n", timeout);
		// 呼び出し元の通常エラー処理へ制御を移す。
		throw ERR_CLIENT_SYSTEM;
	}
	// -1は無制限待ちなので、内部で判別できる特別な期限値を設定する。
	if (timeout == -1)
	{
		// 秒を-1にして、無制限待ちであることを表す。
		deadline.tv_sec = -1;
		// timevalのマイクロ秒部分を初期化する。
		deadline.tv_usec = 0;
		// 無制限待ちの期限を返し、時刻取得による制限を行わない。
		return deadline;
	}
	// 有限時間の期限計算の起点として、現在時刻を取得する。
	if (gettimeofday(&deadline, NULL) < 0)
	{
		// 時刻取得に失敗した原因をログに記録する。
		log_printf(LV_ERROR, "gettimeofday error.(errno = %d)\n", errno);
		// 正しい期限を作れないため、エラーとして処理を中断する。
		throw ERR_CLIENT_SYSTEM;
	}
	// 現在時刻に設定秒数を加えて、処理全体の絶対期限を作る。
	deadline.tv_sec += timeout;

	// 計算した期限を呼び出し元へ返す。
	return deadline;
}

// 目的: select通過後にrecv/sendが停止しても、設定した期限内に処理を戻す。
// 処理: 絶対期限から現在時刻を引いた残り時間を、対象ソケットの受信または送信タイムアウトに設定する。
static void tcpSetSocketTimeout(int sd, int option, const struct timeval &deadline)
{
	// setsockoptへ渡す、ソケット操作の残り待ち時間を格納する。
	struct timeval timeout;
	// 残り時間を計算するための現在時刻を格納する。
	struct timeval now;

	// 特別値-1なら、ソケットの待ち時間制限を無効にする。
	if (deadline.tv_sec == -1)
	{
		// timevalの0はSO_RCVTIMEO/SO_SNDTIMEOで無制限待ちを意味する。
		timeout.tv_sec = 0;
		// マイクロ秒部分も0にして無制限待ちを指定する。
		timeout.tv_usec = 0;
	}
	else
	{
		// 有限期限までの残り時間を求めるため、現在時刻を取得する。
		if (gettimeofday(&now, NULL) < 0)
		{
			// 時刻取得に失敗した原因をログに記録する。
			log_printf(LV_ERROR, "gettimeofday error.(errno = %d)\n", errno);
			// 残り時間を計算できないため、システムエラーとして中断する。
			throw ERR_CLIENT_SYSTEM;
		}
		// 期限と現在時刻の秒差を残り時間の秒部分へ設定する。
		timeout.tv_sec = deadline.tv_sec - now.tv_sec;
		// 期限と現在時刻のマイクロ秒差を残り時間の小数部分へ設定する。
		timeout.tv_usec = deadline.tv_usec - now.tv_usec;
		// マイクロ秒差が負なら、秒から1秒を借りて正規化する。
		if (timeout.tv_usec < 0)
		{
			// 借りた1秒分を秒の残り時間から差し引く。
			timeout.tv_sec--;
			// 借りた1秒をマイクロ秒へ換算して小数部分へ加える。
			timeout.tv_usec += 1000000;
		}
		// 期限超過または残り時間ゼロなら、待ちに入る前にタイムアウトを返す。
		if ((timeout.tv_sec < 0) || ((timeout.tv_sec == 0) && (timeout.tv_usec == 0)))
		{
			// 期限超過をログに記録する。
			log_printf(LV_ERROR, "socket timeout.\n");
			// 既存のselectタイムアウトと同じエラーコードで通知する。
			throw ERR_CLIENT_SELECT_TIMEOUT;
		}
	}

	// 指定された受信/送信オプションへ、計算した残り待ち時間を設定する。
	if (setsockopt(sd, SOL_SOCKET, option, &timeout, sizeof(timeout)) < 0)
	{
		// ソケットオプション設定失敗の内容をログに記録する。
		log_printf(LV_ERROR, "setsockopt timeout error.(option = %d, errno = %d)\n", option, errno);
		// タイムアウトを設定できなかったため、システムエラーとして中断する。
		throw ERR_CLIENT_SYSTEM;
	}
}

/**
 * @brief        コンストラクタ
 */
Client::Client(void)
{
	client_.local.addr_len = sizeof(client_.local.addr);
	client_.remote.addr_len = sizeof(client_.remote.addr);
}

Client::Client(in_addr hostInfo, int portInfo)
{
	client_.remote.addr.sin_family = AF_INET;
	client_.remote.addr.sin_port = portInfo;
	client_.remote.addr.sin_addr = hostInfo;
	client_.local.addr_len = sizeof(client_.local.addr);
	client_.remote.addr_len = sizeof(client_.remote.addr);
}

/**
 * @brief        デストラクタ
 */
Client::~Client()
{
}

/**
 * @brief        初期化処理
 * @param[in]    host        ホスト名
 * @param[in]    service     サービス名
 * @param[out]   hostInfo    IPアドレス情報
 * @param[out]   portInfo    ポート番号情報
 *
 * @retval       なし
 */
void Client::TcpInit(const char *host, const char *service, in_addr *hostInfo, int *portInfo)
{
	log_printf(LV_DEBUG, "start\n");

	try
	{
		// ホスト名からIPアドレスを取得
		if (hostInfo->s_addr == 0)
		{
			tcpGetHostByName(host, hostInfo);
			log_printf(LV_DEBUG, "hostname = %s, hostInfo = %d\n", host, hostInfo->s_addr);
		}

		// サービス名からポート番号を取得
		if (*portInfo == 0)
		{
			tcpGetServiceByName(service, portInfo);
			log_printf(LV_DEBUG, "service = %s, portInfo = %d\n", service, *portInfo);
		}

		client_.remote.addr.sin_family = AF_INET;
		client_.remote.addr.sin_port = *portInfo;
		client_.remote.addr.sin_addr.s_addr = hostInfo->s_addr;

		log_printf(LV_DEBUG, "hostInfo = %d\n", client_.remote.addr.sin_addr.s_addr);
		log_printf(LV_DEBUG, "portInfo = %d\n", client_.remote.addr.sin_port);
		log_printf(LV_DEBUG, "len      = %d\n", client_.remote.addr_len);
	}
	catch (int error)
	{
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		throw error;
	}
	catch (...)
	{
		log_printf(LV_ERROR, "error.\n");
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        サーバ接続
 * @param[inout] sd          ソケットディスクリプタ
 * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
 *
 * @retval       なし
 */
void Client::TcpConnect(int *sd, int connect)
{
	int err = 0;   ///< エラー戻り値
	int sd_ = *sd; ///< ソケットディスクリプタ

	log_printf(LV_DEBUG, "start\n");

	// 常時接続型は初回時のみソケットを生成する
	if ((connect == TCP_CONNECT) && (sd_ >= 0))
	{
		log_printf(LV_DEBUG, "sd(client:after socket) = %d\n", sd_);
		log_printf(LV_DEBUG, "end(no exec.)\n");
		return;
	}

	try
	{
		// ソケットを生成
		sd_ = socket(AF_INET, SOCK_STREAM, 0);
		if (sd_ < 0)
		{
			log_printf(LV_ERROR, "socket error.(sd = %d, errno = %d)\n", sd_, errno);
			throw ERR_CLIENT_SYSTEM;
		}
		log_printf(LV_DEBUG, "sd(client:after socket) = %d\n", sd_);

		// TIME_WAIT状態を回避する
		struct linger optval;
		optval.l_onoff = 1;
		optval.l_linger = 0;
		err = setsockopt(sd_, SOL_SOCKET, SO_LINGER, (void *)&optval, sizeof(optval));
		if (err < 0)
		{
			throw ERR_CLIENT_SYSTEM;
		}

		//SST000-01635:Set tcp_nodelay for changing Nagle mode OFF(for avoid 40msec tcp delay) 
		tcpConnect(sd_);

		// サーバ接続
		err = getsockname(sd_, (struct sockaddr *)&client_.local.addr, (SOCKLEN *)&client_.local.addr_len);
		if (err < 0)
		{
			log_printf(LV_ERROR, "getsockname error.(err = %d, errno = %d)\n", err, errno);
			throw ERR_CLIENT_GETSOCKNAME;
		}
		log_printf(LV_DEBUG, "local.sin_port = %d\n", client_.local.addr.sin_port);
		log_printf(LV_DEBUG, "local.sin_addr = %d\n", client_.local.addr.sin_addr.s_addr);

		// ローカルアドレスを取得
		*sd = sd_;
	}
	catch (int error)
	{
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		// 目的: 接続途中の失敗で、新しく作成したソケットがリークするのを防ぐ。
		// 理由: 接続完了前は新しいFDが*sdへ反映されないため、ローカル変数sd_を使って閉じる。
		// 既存の有効なFDがある場合だけ、今回の接続処理で確保したソケットを後始末する。
		if (sd_ >= 0)
		{
			// 接続途中で確保したソケットを閉じる。
			err = close(sd_);
			// closeに失敗した場合も、後続の例外処理を続けられるよう原因を記録する。
			if (err < 0)
			{
				// 対象FDとOSエラー番号を記録し、クローズ失敗を見えるようにする。
				log_printf(LV_ERROR, "close error.(sd = %d, errno = %d)\n", sd_, errno);
			}
			// 呼び出し元のFDが今回閉じたFDと同じ場合だけ、無効値に更新する。
			if (*sd == sd_)
			{
				// 二重クローズや再利用済みFDへの誤操作を避けるため、FDを無効化する。
				*sd = -1;
			}
		}
		// 元の整数エラーを維持し、呼び出し元のエラー処理へ返す。
		throw error;
	}
	catch (...)
	{
		// 例外の詳細が整数でない場合も記録して後始末する。
		log_printf(LV_ERROR, "error.\n");
		// 目的: 予期しない例外でも、接続途中に作成したソケットを残さない。
		// 理由: 接続完了前は*sdが新しいFDを指さないため、ローカル変数sd_で管理する。
		if (sd_ >= 0)
		{
			// 接続途中で確保したソケットを閉じる。
			err = close(sd_);
			// closeに失敗した場合は、対象FDと原因をログへ残す。
			if (err < 0)
			{
				// クローズ失敗を記録し、例外の再送出は継続する。
				log_printf(LV_ERROR, "close error.(sd = %d, errno = %d)\n", sd_, errno);
			}
			// 呼び出し元のFDが閉じたFDと一致する場合は無効値にする。
			if (*sd == sd_)
			{
				// 後続処理が同じFDを再利用しないよう、-1を設定する。
				*sd = -1;
			}
		}
		// 非整数例外を既存の汎用エラー値へ変換して呼び出し元へ返す。
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        サーバからのデータ読み込み
 *               読み込みは1メッセージ
 * @param[in]    sd          ソケットディスクリプタ
 * @param[out]   size        返信データサイズ
 * @param[out]   data        返信データ
 * @param[out]   error       エラー情報
 * @param[out]   header      ヘッダ(固定文字列)
 * @param[out]   funcId      関数ID
 * @param[out]   parity1     パリティ情報1
 * @param[out]   parity2     パリティ情報2
 *
 * @retval       なし
 */
void Client::TcpRead(int sd, int *size, char **data, int *error, char *header, int *funcId, int *parity1, int *parity2)
{
	log_printf(LV_DEBUG, "start\n");

	// 中間メッセージ、返信メッセージ共にヘッダは8バイトである。
	// 最初に8バイトデータを読み込みヘッダ情報を取得する。
	int headRecvSize = HEADER_SIZE;
	char headData[HEADER_SIZE + 1];
	// 目的: ヘッダだけでなく本文・フッタの受信完了まで時間制限する。
	// 処理: このTcpRead内のすべてのrecvが共有する絶対期限を、受信開始時に一度だけ作る。
	struct timeval deadline = tcpCreateDeadline(RECEIVE_TIMEOUT);
	memset(headData, 0, sizeof(headData));
	// 最初のヘッダ受信について、従来どおり読み込み可能になるまで待つ。
	tcpSelectRfs(sd, RECEIVE_TIMEOUT);
	// ヘッダを受信し、以降の受信にも共通の絶対期限を渡す。
	tcpReadBytes(sd, headRecvSize, headData, deadline);

	if (strncmp(headData, "InfmHead", sizeof(headData)) == 0)
	{

		// 中間メッセージを受信した場合、残りのデータを読み込む
		// 関数ID(4)＋パリティ情報(8)＋エラー情報(4)＋中間フッタ(8)
		S_TcpMsgC msgC;
		memset(&msgC, 0, sizeof(msgC));
		int restRecvSize = sizeof(msgC) - sizeof(msgC.header);
		char *restData = new char[restRecvSize];
		if (restData == NULL)
		{
			throw ERR_CLIENT_MALLOC;
		}
		memset(restData, 0, restRecvSize * sizeof(char));
		tcpReadBytes(sd, restRecvSize, restData, deadline);
		memcpy(&msgC.header, headData, headRecvSize);
		memcpy(&msgC.funcId, restData, restRecvSize);
		if (restData != NULL)
		{
			delete[] restData;
			restData = NULL;
		}
		memcpy(header, headData, headRecvSize);
		*funcId = ntohl(msgC.funcId);
		*parity1 = ntohl(msgC.parity1);
		*parity2 = ntohl(msgC.parity2);
		*error = ntohl(msgC.error);
	}
	else if (strncmp(headData, "RecvHead", sizeof(headData)) == 0)
	{

		// 返信メッセージを受信した場合
		// 関数ID(4)＋パリティ情報(8)＋データサイズ(4)＋エラー情報(4)＋返信データ(可変)＋返信フッタ(8)
		// 1. エラー情報(4)までのデータを読み込む
		S_TcpMsgE msgE;
		memset(&msgE, 0, sizeof(msgE));
		int fixRecvSize = sizeof(msgE.funcId) + sizeof(msgE.parity1) + sizeof(msgE.parity2) + sizeof(msgE.size) + sizeof(msgE.error);
		char *fixData = new char[fixRecvSize];
		if (fixData == NULL)
		{
			throw ERR_CLIENT_MALLOC;
		}
		memset(fixData, 0, fixRecvSize * sizeof(char));
		tcpReadBytes(sd, fixRecvSize, fixData, deadline);
		memcpy(&msgE.header, headData, headRecvSize);
		memcpy(&msgE.funcId, fixData, fixRecvSize);
		if (fixData != NULL)
		{
			delete[] fixData;
			fixData = NULL;
		}

		// 2. 取得したデータサイズの情報を元に残りのデータを読み込む
		msgE.size = ntohl(msgE.size);
		if (msgE.size <= 0)
		{
			log_printf(LV_DEBUG, "parameter size is illegal.(%d)\n", msgE.size);
			throw ERR_CLIENT_RECEIVE;
		}
		*size = msgE.size;
		int restRecvSize = *size + sizeof(msgE.footer);
		if (*data != NULL)
		{
			delete[] *data;
			*data = NULL;
		}
		*data = new char[*size];
		char *restData = new char[restRecvSize];
		if ((*data == NULL) || (restData == NULL))
		{
			throw ERR_CLIENT_MALLOC;
		}
		memset(*data, 0, *size * sizeof(char));
		memset(restData, 0, restRecvSize * sizeof(char));
		tcpReadBytes(sd, restRecvSize, restData, deadline);
		memcpy(*data, &restData[0], *size);
		memcpy(&msgE.footer, &restData[*size], sizeof(msgE.footer));
		if (restData != NULL)
		{
			// dataの領域解放は通常上位で行う
			delete[] restData;
			restData = NULL;
		}
		memcpy(header, headData, headRecvSize);
		*funcId = ntohl(msgE.funcId);
		*parity1 = ntohl(msgE.parity1);
		*parity2 = ntohl(msgE.parity2);
		*error = ntohl(msgE.error);
	}
	else
	{
		// ヘッダが見つからない
		log_printf(LV_ERROR, "not find header error.(%s)\n", headData);
		throw ERR_CLIENT_RECEIVE;
	}

	log_printf(LV_DEBUG, "header  = %s\n", header);
	log_printf(LV_DEBUG, "funcId  = %d\n", *funcId);
	log_printf(LV_DEBUG, "parity1 = %d\n", *parity1);
	log_printf(LV_DEBUG, "parity2 = %d\n", *parity2);
	log_printf(LV_DEBUG, "error   = %d\n", *error);

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        サーバにデータ書き込み
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    size        送信データサイズ
 * @param[in]    data        送信データ
 *
 * @retval       なし
 */
void Client::TcpWrite(int sd, int size, char *data)
{
	log_printf(LV_DEBUG, "start\n");

	// サーバにデータ書き込み
	// 目的: 送信が部分送信に分かれても、TcpWrite全体の待ち時間に上限を設ける。
	// 処理: select待ちと後続の全sendで共有する絶対期限を、送信開始時に一度だけ作る。
	struct timeval deadline = tcpCreateDeadline(SEND_TIMEOUT);
	// 最初の送信前に、ソケットが書き込み可能になるまで設定時間だけ待つ。
	tcpSelectWfs(sd, SEND_TIMEOUT);
	// 残りの全データを送信し、各sendにも同じ絶対期限を適用する。
	tcpWriteBytes(sd, size, data, deadline);

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        サーバ切断
 * @param[inout] sd          ソケットディスクリプタ
 * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
 *
 * @retval       なし
 */
void Client::TcpClose(int *sd, int connect)
{
	int err = 0;   ///< エラー戻り値
	int sd_ = *sd; ///< ソケットディスクリプタ

	log_printf(LV_DEBUG, "start\n");

	// 常時接続型は切断しない
	if (connect == TCP_CONNECT)
	{
		log_printf(LV_DEBUG, "end(no exec.)\n");
		return;
	}

	// 書き込みを閉じ、相手にFINを送る
	// 相手からのFINを待たずに強制終了する
	err = shutdown(sd_, SHUT_RDWR);
	if (err < 0)
	{
		log_printf(LV_ERROR, "shutdown error.(err = %d, errno = %d)\n", err, errno);
	}

	// ソケットを閉じる
	err = close(sd_);
	if (err < 0)
	{
		log_printf(LV_ERROR, "close error.(err = %d, errno = %d)\n", err, errno);
	}
	log_printf(LV_DEBUG, "sd(client:after close) = %d\n", *sd);
	*sd = -1;

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        ホスト名からIPアドレスを取得
 * @param[in]    host        ホスト名
 * @param[out]   in_addr     IPアドレス(struct in_addr)
 *
 * @retval       なし
 */
void Client::tcpGetHostByName(const char *host, struct in_addr *in_addr)
{
	struct hostent *h_info; ///< hostent構造体

	// ホスト名からIPアドレスを取得
	h_info = gethostbyname(host);
	if (h_info == NULL)
	{
		log_printf(LV_ERROR, "gethostbyname error.(%d)\n", errno);
		throw ERR_CLIENT_GETHOSTBYNAME;
	}
	memcpy(&in_addr->s_addr, h_info->h_addr_list[0], h_info->h_length);
	log_printf(LV_DEBUG, "h_length = %d\n", h_info->h_length);
	log_printf(LV_DEBUG, "s_addr = %d\n", (int)in_addr->s_addr);

	return;
}

/**
 * @brief        サービス名からポート番号を取得
 * @param[in]    service     サービス名
 * @param[out]   port        ポート番号
 *
 * @retval       なし
 */
void Client::tcpGetServiceByName(const char *service, int *port)
{
	struct servent *se = NULL; ///< servent構造体

	// サービス名からポート番号を取得
	// プロトコルはTCP固定
	se = getservbyname(service, "tcp");
	if (se == NULL)
	{
		log_printf(LV_ERROR, "getservbyname error.(%d)\n", errno);
		throw ERR_CLIENT_GETSERVBYNAME;
	}
	*port = se->s_port;
	log_printf(LV_DEBUG, "service: %s, port:%d\n", service, *port);

	return;
}

/**
 * @brief        connect実行
 * @param[in]    sd          ソケットディスクリプタ
 *
 * @retval       なし
 */
void Client::tcpConnect(int sd)
{
	int err = 0;		   ///< エラー戻り値
	int connect_errno = 0; ///< 接続時のエラー情報

	log_printf(LV_DEBUG, "start\n");

	try
	{
		// ノンブロッキングモードに変更
		tcpSetNonBlockingMode(sd);

		// サーバ接続
		err = connect(sd, (struct sockaddr *)&(client_.remote.addr), client_.remote.addr_len);
		connect_errno = errno;
		if (err == 0)
		{
			// サーバとクライアントが同一ホストの場合はこのタイミングでコネクションが確立するはず
			// ブロッキングモードに戻す
			tcpSetBlockingMode(sd);
			log_printf(LV_DEBUG, "end\n");
			return;
		}
		if (connect_errno != EINPROGRESS)
		{
			// サーバとクライアントが別ホストの場合は、コネクション確立を開始していればEINPROGRESSが返ってくるはず
			log_printf(LV_ERROR, "connect error = %d\n", connect_errno);
			throw ERR_CLIENT_CONNECT;
		}

		// select実行
		tcpSelect(sd, CONNECT_TIMEOUT);

		// ブロッキングモードに戻す
		tcpSetBlockingMode(sd);
	}
	catch (int error)
	{
		// sdが取れた場合のみブロッキングモードに戻す
		if (error != ERR_CLIENT_19)
		{
			tcpSetBlockingMode(sd);
		}
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		throw error;
	}
	catch (...)
	{
		tcpSetBlockingMode(sd);
		log_printf(LV_ERROR, "error.\n");
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        select実行(読み込み可能待ち)
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    timeout     タイムアウト時間 [sec]
 *
 * @retval       なし
 */
void Client::tcpSelectRfs(int sd, int timeout)
{
	int err = 0;	   ///< エラー戻り値
	fd_set rfs;		   ///< ファイルディスクリプタ
	struct timeval tv; ///< タイムアウト時間設定

	log_printf(LV_DEBUG, "start\n");

	try
	{
		FD_ZERO(&rfs);
		FD_SET(sd, &rfs);

		// セレクトのタイムアウト時間を設定
		tv.tv_sec = timeout;
		tv.tv_usec = 0;

		// select()実行
		if (timeout == -1)
		{
			err = select(sd + 1, &rfs, NULL, NULL, NULL); // タイムアウトなし
		}
		else
		{
			err = select(sd + 1, &rfs, NULL, NULL, &tv);
		}
		if (err == 0)
		{
			// タイムアウト
			log_printf(LV_DEBUG, "select timeout.(%d)\n", (int)tv.tv_sec);
			throw ERR_CLIENT_SELECT_TIMEOUT;
		}
		else if (err < 0)
		{
			log_printf(LV_ERROR, "select error.(err = %d, errno = %d)\n", err, errno);
			throw ERR_CLIENT_SELECT;
		}

		// ファイルディスクリプタにデータあり
		if (FD_ISSET(sd, &rfs))
		{
			int error_code;
			int len = sizeof(error_code);
			;

			// ソケット中の保留のエラーを取得
			err = getsockopt(sd, SOL_SOCKET, SO_ERROR, &error_code, (SOCKLEN *)&len);
			if (err < 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(err = %d, errno = %d)\n", err, errno);
				throw ERR_CLIENT_GETSOCKOPT;
			}

			// 念のためエラーチェック
			if (error_code != 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(%d)\n", error_code);
				throw ERR_CLIENT_GETSOCKOPT_2;
			}
		}
		else
		{
			log_printf(LV_DEBUG, "socket not set.\n");
			throw ERR_CLIENT_SELECT_NO_SOCKET;
		}
	}
	catch (int error)
	{
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		throw error;
	}
	catch (...)
	{
		log_printf(LV_ERROR, "error.\n");
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        select実行(書き込み可能待ち)
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    timeout     タイムアウト時間 [sec]
 *
 * @retval       なし
 */
void Client::tcpSelectWfs(int sd, int timeout)
{
	int err = 0;	   ///< エラー戻り値
	fd_set wfs;		   ///< ファイルディスクリプタ
	struct timeval tv; ///< タイムアウト時間設定

	log_printf(LV_DEBUG, "start\n");

	try
	{
		FD_ZERO(&wfs);
		FD_SET(sd, &wfs);

		// セレクトのタイムアウト時間を設定
		tv.tv_sec = timeout;
		tv.tv_usec = 0;

		// select()実行
		if (timeout == -1)
		{
			err = select(sd + 1, NULL, &wfs, NULL, NULL); // タイムアウトなし
		}
		else
		{
			err = select(sd + 1, NULL, &wfs, NULL, &tv);
		}
		if (err == 0)
		{
			// タイムアウト
			log_printf(LV_DEBUG, "select timeout.(%d)\n", (int)tv.tv_sec);
			throw ERR_CLIENT_SELECT_TIMEOUT;
		}
		else if (err < 0)
		{
			log_printf(LV_ERROR, "select error.(err = %d, errno = %d)\n", err, errno);
			throw ERR_CLIENT_SELECT;
		}

		// ファイルディスクリプタにデータあり
		if (FD_ISSET(sd, &wfs))
		{
			int error_code;
			int len = sizeof(error_code);
			;

			// ソケット中の保留のエラーを取得
			err = getsockopt(sd, SOL_SOCKET, SO_ERROR, &error_code, (SOCKLEN *)&len);
			if (err < 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(err = %d, errno = %d)\n", err, errno);
				throw ERR_CLIENT_GETSOCKOPT;
			}

			// 念のためエラーチェック
			if (error_code != 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(%d)\n", error_code);
				throw ERR_CLIENT_GETSOCKOPT_2;
			}
		}
		else
		{
			log_printf(LV_DEBUG, "socket not set.\n");
			throw ERR_CLIENT_SELECT_NO_SOCKET;
		}
	}
	catch (int error)
	{
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		throw error;
	}
	catch (...)
	{
		log_printf(LV_ERROR, "error.\n");
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        select実行(読み書き可能待ち)
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    timeout     タイムアウト時間 [sec]
 *
 * @retval       なし
 */
void Client::tcpSelect(int sd, int timeout)
{
	int err = 0;	   ///< エラー戻り値
	fd_set rfs, wfs;   ///< ファイルディスクリプタ
	struct timeval tv; ///< タイムアウト時間設定

	log_printf(LV_DEBUG, "start\n");

	try
	{
		// 読み書き両方をセレクトで待つ設定
		FD_ZERO(&rfs);
		FD_SET(sd, &rfs);
		FD_ZERO(&wfs);
		FD_SET(sd, &wfs);

		// セレクトのタイムアウト時間を設定
		tv.tv_sec = timeout;
		tv.tv_usec = 0;

		// select()実行
		if (timeout == -1)
		{
			err = select(sd + 1, &rfs, &wfs, NULL, NULL); // タイムアウトなし
		}
		else
		{
			err = select(sd + 1, &rfs, &wfs, NULL, &tv);
		}
		if (err == 0)
		{
			// タイムアウト
			log_printf(LV_DEBUG, "select timeout.(%d)\n", (int)tv.tv_sec);
			throw ERR_CLIENT_SELECT_TIMEOUT;
		}
		else if (err < 0)
		{
			log_printf(LV_ERROR, "select error.(err = %d, errno = %d)\n", err, errno);
			throw ERR_CLIENT_SELECT;
		}

		// 読み書きができれば接続完了
		if (FD_ISSET(sd, &rfs) || FD_ISSET(sd, &wfs))
		{
			int error_code;
			int len = sizeof(error_code);
			;

			// ソケット中の保留のエラーを取得
			err = getsockopt(sd, SOL_SOCKET, SO_ERROR, &error_code, (SOCKLEN *)&len);
			if (err < 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(err = %d, errno = %d)\n", err, errno);
				throw ERR_CLIENT_GETSOCKOPT;
			}

			// 念のためエラーチェック
			if (error_code != 0)
			{
				log_printf(LV_ERROR, "getsockopt error.(%d)\n", error_code);
				throw ERR_CLIENT_GETSOCKOPT_2;
			}
		}
		else
		{
			log_printf(LV_DEBUG, "socket not set.\n");
			throw ERR_CLIENT_SELECT_NO_SOCKET;
		}
	}
	catch (int error)
	{
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		throw error;
	}
	catch (...)
	{
		log_printf(LV_ERROR, "error.\n");
		throw -1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        ノンブロッキングモードに変更
 * @param[in]    sd          ソケットディスクリプタ
 *
 * @retval       なし
 */
void Client::tcpSetNonBlockingMode(int sd)
{
	int flags; ///< fcntl戻り値

	flags = fcntl(sd, F_GETFL, 0);
	if (flags < 0)
	{
		log_printf(LV_ERROR, "fcntl error.(err = %d, errno = %d)\n", flags, errno);
		throw ERR_CLIENT_FCNTL;
	}

	if (fcntl(sd, F_SETFL, flags | O_NONBLOCK))
	{
		log_printf(LV_ERROR, "fcntl error.(errno = %d)\n", errno);
		throw ERR_CLIENT_FCNTL;
	}

	return;
}

/**
 * @brief        ブロッキングモードに変更
 * @param[in]    sd          ソケットディスクリプタ
 *
 * @retval       なし
 */
void Client::tcpSetBlockingMode(int sd)
{
	int flags; ///< fcntl戻り値

	flags = fcntl(sd, F_GETFL, 0);
	if (flags < 0)
	{
		log_printf(LV_ERROR, "fcntl error.(err = %d, errno = %d)\n", flags, errno);
		throw ERR_CLIENT_FCNTL;
	}

	if (fcntl(sd, F_SETFL, flags & ~O_NONBLOCK))
	{
		log_printf(LV_ERROR, "fcntl error.(errno = %d)\n", errno);
		throw ERR_CLIENT_FCNTL;
	}

	return;
}

/**
 * @brief        指定したバイトだけデータを受信
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    size        データサイズ
 * @param[out]   data        受信データ
 *
 * @retval       なし
 */
void Client::tcpReadBytes(int sd, int size, char *data, const struct timeval &deadline)
{
	int restSize = 0; ///< 残受信データサイズ
	int nRecv = 0;	  ///< データ受信結果

	log_printf(LV_DEBUG, "start\n");

	// 指定サイズ受信できるまで繰り返し受信を行う
	restSize = size;
	while (restSize > 0)
	{

		log_printf(LV_DEBUG, "recv size(%d)\n", restSize);
		// このrecvが期限を越えてブロックしないよう、受信直前に残り時間を設定する。
		tcpSetSocketTimeout(sd, SO_RCVTIMEO, deadline);
		// ソケットから未受信分を読み込み、実際に受け取ったバイト数を得る。
		nRecv = recv(sd, data, restSize, 0);
		// recvがエラーを返した場合は、タイムアウトか通常の受信エラーかを判定する。
		if (nRecv < 0)
		{
			// ソケット受信タイムアウトを示すエラー番号か確認する。
			if ((errno == EAGAIN) || (errno == EWOULDBLOCK))
			{
				// 受信タイムアウトをログに記録し、既存のタイムアウトエラーとして通知する。
				log_printf(LV_ERROR, "recv timeout.\n");
				// 呼び出し元の共通タイムアウト処理へ制御を移す。
				throw ERR_CLIENT_SELECT_TIMEOUT;
			}
			// 受信エラー
			log_printf(LV_ERROR, "recv error.(err = %d, errno = %d)\n", nRecv, errno);
			throw ERR_CLIENT_RECV;
		}
		else if (nRecv == 0)
		{
			// 回線切断
			log_printf(LV_ERROR, "connection closed error.\n");
			throw ERR_CLIENT_RECV_DISCONNECT;
		}
		data += nRecv;
		restSize -= nRecv;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        指定したバイトだけデータを送信
 * @param[in]    sd          ソケットディスクリプタ
 * @param[in]    size        データサイズ
 * @param[in]    data        送信データ
 *
 * @retval       なし
 */
void Client::tcpWriteBytes(int sd, int size, char *data, const struct timeval &deadline)
{
	int restSize = 0; ///< 残送信データサイズ
	int nSend = 0;	  ///< データ送信結果

	log_printf(LV_DEBUG, "start\n");

	restSize = size;
	while (restSize > 0)
	{

		// SIGPIPEの発生を抑止
		log_printf(LV_DEBUG, "send size(%d)\n", restSize);
		// このsendが期限を越えてブロックしないよう、送信直前に残り時間を設定する。
		tcpSetSocketTimeout(sd, SO_SNDTIMEO, deadline);
		// 未送信分をソケットへ書き込み、実際に送れたバイト数を得る。
		nSend = send(sd, data, restSize, 0);
		// sendがエラーを返した場合は、タイムアウトか通常の送信エラーかを判定する。
		if (nSend < 0)
		{
			// ソケット送信タイムアウトを示すエラー番号か確認する。
			if ((errno == EAGAIN) || (errno == EWOULDBLOCK))
			{
				// 送信タイムアウトをログに記録し、既存のタイムアウトエラーとして通知する。
				log_printf(LV_ERROR, "send timeout.\n");
				// 呼び出し元の共通タイムアウト処理へ制御を移す。
				throw ERR_CLIENT_SELECT_TIMEOUT;
			}
			// 送信エラー
			log_printf(LV_ERROR, "send error.(err = %d, errno = %d)\n", nSend, errno);
			throw ERR_CLIENT_SEND;
		}
		else if (nSend == 0)
		{
			// 回線切断
			log_printf(LV_ERROR, "connection closed error.\n");
			throw ERR_CLIENT_SEND_DISCONNECT;
		}
		data += nSend;
		restSize -= nSend;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}


----------------------------------------------------------------------------------------------------
Client.hpp
----------------------------------------------------------------------------------------------------
/**
 * @file    Client.hpp
 * @author  Copyright(C) Canon Inc. All right reserved.
 * @date    2014/01/10
 * @brief   クライアントクラス
 * @par     説明
 *  - TCP通信のクライアント機能を実現するクラス
 */
#ifndef __Client_h__
#define __Client_h__

#ifdef __cplusplus
extern "C" {
#endif
#include <netdb.h>
#ifdef __cplusplus
}
#endif

#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <string>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <errno.h>
#ifdef LINUX
#include <sys/time.h>
#else
#include <time.h>
#endif

/**
 * @brief ChorusとLinuxの差分吸収
 */
#ifdef LINUX
#define SOCKLEN socklen_t       ///< アドレスの長さを格納する型
#else
#define SOCKLEN int             ///< アドレスの長さを格納する型
#endif

/**
 * @brief エラーstatus2
 */
static const int ERR_CLIENT_TIMEOUT  = 0x1;
static const int ERR_CLIENT_RECEIVE  = 0x2;
static const int ERR_CLIENT_SYSTEM   = 0x20;
static const int ERR_CLIENT_MALLOC   = 0x30;
static const int ERR_CLIENT_GETSOCKNAME   = 0x41;
static const int ERR_CLIENT_GETHOSTBYNAME   = 0x42;
static const int ERR_CLIENT_GETSERVBYNAME   = 0x43;
static const int ERR_CLIENT_CONNECT   = 0x44;
static const int ERR_CLIENT_SELECT_TIMEOUT   = 0x45;
static const int ERR_CLIENT_SELECT   = 0x46;
static const int ERR_CLIENT_GETSOCKOPT   = 0x47;
static const int ERR_CLIENT_GETSOCKOPT_2   = 0x48;
static const int ERR_CLIENT_SELECT_NO_SOCKET   = 0x49;
static const int ERR_CLIENT_FCNTL  = 0x4a;
static const int ERR_CLIENT_RECV  = 0x4b;
static const int ERR_CLIENT_RECV_DISCONNECT  = 0x4c;
static const int ERR_CLIENT_SEND  = 0x4d;
static const int ERR_CLIENT_SEND_DISCONNECT  = 0x4e;
static const int ERR_CLIENT_15  = 0x4f;
static const int ERR_CLIENT_16  = 0x50;
static const int ERR_CLIENT_17  = 0x51;
static const int ERR_CLIENT_18  = 0x52;
static const int ERR_CLIENT_19  = 0x53;
static const int ERR_CLIENT_21  = 0x55;
static const int ERR_CLIENT_22  = 0x56;
static const int ERR_CLIENT_23  = 0x57;


/**
 * @brief ローカル定義
 */
const int CONNECT_TIMEOUT        = 30; ///< Connect timeout in seconds; -1 disables it.
const int TIMEOUT        = 3000; ///< タイムアウト時間[sec]
const int HEADER_SIZE    = 8;   ///< ヘッダサイズ(8バイト固定)

#ifndef LINUX
/**
 * @brief shutdownモード
 */
typedef enum {
    SHUT_RD = 0,               ///< Read
    SHUT_WR,                   ///< Write
    SHUT_RDWR,                 ///< Read & Write
} E_ShutdownMode;
#else
#endif

/**
 * @brief TCP構造体
 */
typedef struct {
    struct sockaddr_in addr;     ///< アドレス
    size_t             addr_len; ///< アドレス長
} S_TcpAddress;

/**
 * @brief TCPクライアント情報
 */
typedef struct {
    S_TcpAddress local;          ///< ローカル(Client側)アドレス情報
    S_TcpAddress remote;         ///< リモート(Server側)アドレス情報
} S_TcpClient;

/**
 * @brief 送信メッセージ固定部フォーマット
 */
typedef struct {
    char         header[8];      ///< 返信ヘッダ(固定文字列)
    int          funcId;         ///< 関数ID
    int          parity1;        ///< パリティ情報1
    int          parity2;        ///< パリティ情報2
    int          size;           ///< データサイズ
    int          cMsg;           ///< 中間メッセージ有無情報(0:なし/1:あり)
    char         footer[8];      ///< 返信フッタ(固定文字列)
} S_TcpMsgS;

/**
 * @brief 中間メッセージ固定部フォーマット
 */
typedef struct {
    char         header[8];      ///< 返信ヘッダ(固定文字列)
    int          funcId;         ///< 関数ID
    int          parity1;        ///< パリティ情報1
    int          parity2;        ///< パリティ情報2
    int          error;          ///< エラー情報
    char         footer[8];      ///< 返信フッタ(固定文字列)
} S_TcpMsgC;

/**
 * @brief 返信メッセージ固定部フォーマット
 */
typedef struct {
    char         header[8];      ///< 返信ヘッダ(固定文字列)
    int          funcId;         ///< 関数ID
    int          parity1;        ///< パリティ情報1
    int          parity2;        ///< パリティ情報2
    int          size;           ///< データサイズ
    int          error;          ///< エラー情報
    char         footer[8];      ///< 返信フッタ(固定文字列)
} S_TcpMsgE;

class Client
{
public:
    /**
     * @brief        コンストラクタ
     */
    Client(void);
    Client(in_addr hostInfo, int portInfo);

    /**
     * @brief        デストラクタ
     */
    ~Client();

    /**
     * @brief        初期化処理
     * @param[in]    host        ホスト名
     * @param[in]    service     サービス名
     * @param[out]   hostInfo    IPアドレス情報
     * @param[out]   portInfo    ポート番号情報
     */
    void TcpInit(const char *host, const char *service, in_addr *hostInfo, int *portInfo);

    /**
     * @brief        サーバ接続
     * @param[inout] sd          ソケットディスクリプタ
     * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
     */
    void TcpConnect(int *sd, int connect);

    /**
     * @brief        サーバからのデータ読み込み
     *               読み込みは1メッセージ
     * @param[in]    sd          ソケットディスクリプタ
     * @param[out]   size        返信データサイズ
     * @param[out]   data        返信データ
     * @param[out]   error       エラー情報
     * @param[out]   header      ヘッダ(固定文字列)
     * @param[out]   funcId      関数ID
     * @param[out]   parity1     パリティ情報1
     * @param[out]   parity2     パリティ情報2
     */
    void TcpRead(int sd, int *size, char **data, int *error, char *header, int *funcId, int *parity1, int *parity2);

    /**
     * @brief        サーバにデータ書き込み
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    size        送信データサイズ
     * @param[in]    data        送信データ
     */
    void TcpWrite(int sd, int size, char *data);


    /**
     * @brief        サーバ切断
     * @param[inout] sd          ソケットディスクリプタ
     * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
     */
    void TcpClose(int *sd, int connect);

private:
    S_TcpClient client_;     ///< TCPクライアント情報

    /**
     * @brief        ホスト名からIPアドレスを取得
     * @param[in]    host        ホスト名
     * @param[out]   in_addr     IPアドレス(struct in_addr)
     */
    void tcpGetHostByName(const char *host, struct in_addr *in_addr);

    /**
     * @brief        サービス名からポート番号を取得
     * @param[in]    service     サービス名
     * @param[out]   port        ポート番号
     */
    void tcpGetServiceByName(const char *service, int *port);

    /**
     * @brief        connect実行
     * @param[in]    sd          ソケットディスクリプタ
     */
    void tcpConnect(int sd);

    /**
     * @brief        select実行(読み込み可能待ち)
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    timeout     タイムアウト時間 [sec]
     */
    void tcpSelectRfs(int sd, int timeout);

    /**
     * @brief        select実行(書き込み可能待ち)
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    timeout     タイムアウト時間 [sec]
     */
    void tcpSelectWfs(int sd, int timeout);

    /**
     * @brief        select実行(読み書き可能待ち)
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    timeout     タイムアウト時間 [sec]
     */
    void tcpSelect(int sd, int timeout);

    /**
     * @brief        ノンブロッキングモードに変更
     * @param[in]    sd          ソケットディスクリプタ
     */
    void tcpSetNonBlockingMode(int sd);

    /**
     * @brief        ブロッキングモードに変更
     * @param[in]    sd          ソケットディスクリプタ
     */
    void tcpSetBlockingMode(int sd);

    /**
     * @brief        指定したバイトだけデータを受信
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    size        データサイズ
     * @param[out]   data        受信データ
     */
    void tcpReadBytes(int sd, int size, char *data);

    /**
     * @brief        指定したバイトだけデータを送信
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    size        データサイズ
     * @param[in]    data        送信データ
     */
    void tcpWriteBytes(int sd, int size, char *data);
};

#endif // __Client_h__


----------------------------------------------------------------------------------------------------
/**
 * @file    Client.hpp
 * @author  Copyright(C) Canon Inc. All right reserved.
 * @date    2014/01/10
 * @brief   クライアントクラス
 * @par     説明
 *  - TCP通信のクライアント機能を実現するクラス
 */
#ifndef __Client_h__
#define __Client_h__

#ifdef __cplusplus
extern "C" {
#endif
#include <netdb.h>
#ifdef __cplusplus
}
#endif

#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <string>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <errno.h>
#ifdef LINUX
#include <sys/time.h>
#else
#include <time.h>
#endif

/**
 * @brief ChorusとLinuxの差分吸収
 */
#ifdef LINUX
#define SOCKLEN socklen_t       ///< アドレスの長さを格納する型
#else
#define SOCKLEN int             ///< アドレスの長さを格納する型
#endif

/**
 * @brief エラーstatus2
 */
static const int ERR_CLIENT_TIMEOUT  = 0x1;
static const int ERR_CLIENT_RECEIVE  = 0x2;
static const int ERR_CLIENT_SYSTEM   = 0x20;
static const int ERR_CLIENT_MALLOC   = 0x30;
static const int ERR_CLIENT_GETSOCKNAME   = 0x41;
static const int ERR_CLIENT_GETHOSTBYNAME   = 0x42;
static const int ERR_CLIENT_GETSERVBYNAME   = 0x43;
static const int ERR_CLIENT_CONNECT   = 0x44;
static const int ERR_CLIENT_SELECT_TIMEOUT   = 0x45;
static const int ERR_CLIENT_SELECT   = 0x46;
static const int ERR_CLIENT_GETSOCKOPT   = 0x47;
static const int ERR_CLIENT_GETSOCKOPT_2   = 0x48;
static const int ERR_CLIENT_SELECT_NO_SOCKET   = 0x49;
static const int ERR_CLIENT_FCNTL  = 0x4a;
static const int ERR_CLIENT_RECV  = 0x4b;
static const int ERR_CLIENT_RECV_DISCONNECT  = 0x4c;
static const int ERR_CLIENT_SEND  = 0x4d;
static const int ERR_CLIENT_SEND_DISCONNECT  = 0x4e;
static const int ERR_CLIENT_15  = 0x4f;
static const int ERR_CLIENT_16  = 0x50;
static const int ERR_CLIENT_17  = 0x51;
static const int ERR_CLIENT_18  = 0x52;
static const int ERR_CLIENT_19  = 0x53;
static const int ERR_CLIENT_21  = 0x55;
static const int ERR_CLIENT_22  = 0x56;
static const int ERR_CLIENT_23  = 0x57;


/**
 * @brief ローカル定義
 */
const int CONNECT_TIMEOUT        = 30;   ///< 接続待ちの上限時間[秒]。-1は無制限。
const int RECEIVE_TIMEOUT        = 3000; ///< 1回のTcpRead全体の受信上限時間[秒]。-1は無制限。
const int SEND_TIMEOUT           = 3000; ///< 1回のTcpWrite全体の送信上限時間[秒]。-1は無制限。
const int TIMEOUT                = RECEIVE_TIMEOUT; ///< 従来コードとの互換用。受信タイムアウトを参照。
const int HEADER_SIZE    = 8;   ///< ヘッダサイズ(8バイト固定)

#ifndef LINUX
/**
 * @brief shutdownモード
 */
typedef enum {
    SHUT_RD = 0,               ///< Read
    SHUT_WR,                   ///< Write
    SHUT_RDWR,                 ///< Read & Write
} E_ShutdownMode;
#else
#endif

/**
 * @brief TCP構造体
 */
typedef struct {
    struct sockaddr_in addr;     ///< アドレス
    size_t             addr_len; ///< アドレス長
} S_TcpAddress;

/**
 * @brief TCPクライアント情報
 */
typedef struct {
    S_TcpAddress local;          ///< ローカル(Client側)アドレス情報
    S_TcpAddress remote;         ///< リモート(Server側)アドレス情報
} S_TcpClient;

/**
 * @brief 送信メッセージ固定部フォーマット
 */
typedef struct {
    char         header[8];      ///< 返信ヘッダ(固定文字列)
    int          funcId;         ///< 関数ID
    int          parity1;        ///< パリティ情報1
    int          parity2;        ///< パリティ情報2
    int          size;           ///< データサイズ
    int          cMsg;           ///< 中間メッセージ有無情報(0:なし/1:あり)
    char         footer[8];      ///< 返信フッタ(固定文字列)
} S_TcpMsgS;

/**
 * @brief 中間メッセージ固定部フォーマット
 */
typedef struct {
    char         header[8];      ///< 返信ヘッダ(固定文字列)
    int          funcId;         ///< 関数ID
    int          parity1;        ///< パリティ情報1
    int          parity2;        ///< パリティ情報2
    int          error;          ///< エラー情報
    char         footer[8];      ///< 返信フッタ(固定文字列)
} S_TcpMsgC;

/**
 * @brief 返信メッセージ固定部フォーマット
 */
typedef struct {
    char         header[8];      ///< 返信ヘッダ(固定文字列)
    int          funcId;         ///< 関数ID
    int          parity1;        ///< パリティ情報1
    int          parity2;        ///< パリティ情報2
    int          size;           ///< データサイズ
    int          error;          ///< エラー情報
    char         footer[8];      ///< 返信フッタ(固定文字列)
} S_TcpMsgE;

class Client
{
public:
    /**
     * @brief        コンストラクタ
     */
    Client(void);
    Client(in_addr hostInfo, int portInfo);

    /**
     * @brief        デストラクタ
     */
    ~Client();

    /**
     * @brief        初期化処理
     * @param[in]    host        ホスト名
     * @param[in]    service     サービス名
     * @param[out]   hostInfo    IPアドレス情報
     * @param[out]   portInfo    ポート番号情報
     */
    void TcpInit(const char *host, const char *service, in_addr *hostInfo, int *portInfo);

    /**
     * @brief        サーバ接続
     * @param[inout] sd          ソケットディスクリプタ
     * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
     */
    void TcpConnect(int *sd, int connect);

    /**
     * @brief        サーバからのデータ読み込み
     *               読み込みは1メッセージ
     * @param[in]    sd          ソケットディスクリプタ
     * @param[out]   size        返信データサイズ
     * @param[out]   data        返信データ
     * @param[out]   error       エラー情報
     * @param[out]   header      ヘッダ(固定文字列)
     * @param[out]   funcId      関数ID
     * @param[out]   parity1     パリティ情報1
     * @param[out]   parity2     パリティ情報2
     */
    void TcpRead(int sd, int *size, char **data, int *error, char *header, int *funcId, int *parity1, int *parity2);

    /**
     * @brief        サーバにデータ書き込み
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    size        送信データサイズ
     * @param[in]    data        送信データ
     */
    void TcpWrite(int sd, int size, char *data);


    /**
     * @brief        サーバ切断
     * @param[inout] sd          ソケットディスクリプタ
     * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
     */
    void TcpClose(int *sd, int connect);

private:
    S_TcpClient client_;     ///< TCPクライアント情報

    /**
     * @brief        ホスト名からIPアドレスを取得
     * @param[in]    host        ホスト名
     * @param[out]   in_addr     IPアドレス(struct in_addr)
     */
    void tcpGetHostByName(const char *host, struct in_addr *in_addr);

    /**
     * @brief        サービス名からポート番号を取得
     * @param[in]    service     サービス名
     * @param[out]   port        ポート番号
     */
    void tcpGetServiceByName(const char *service, int *port);

    /**
     * @brief        connect実行
     * @param[in]    sd          ソケットディスクリプタ
     */
    void tcpConnect(int sd);

    /**
     * @brief        select実行(読み込み可能待ち)
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    timeout     タイムアウト時間 [sec]
     */
    void tcpSelectRfs(int sd, int timeout);

    /**
     * @brief        select実行(書き込み可能待ち)
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    timeout     タイムアウト時間 [sec]
     */
    void tcpSelectWfs(int sd, int timeout);

    /**
     * @brief        select実行(読み書き可能待ち)
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    timeout     タイムアウト時間 [sec]
     */
    void tcpSelect(int sd, int timeout);

    /**
     * @brief        ノンブロッキングモードに変更
     * @param[in]    sd          ソケットディスクリプタ
     */
    void tcpSetNonBlockingMode(int sd);

    /**
     * @brief        ブロッキングモードに変更
     * @param[in]    sd          ソケットディスクリプタ
     */
    void tcpSetBlockingMode(int sd);

    /**
     * @brief        指定したバイトだけデータを受信
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    size        データサイズ
     * @param[out]   data        受信データ
     */
    void tcpReadBytes(int sd, int size, char *data, const struct timeval& deadline);

    /**
     * @brief        指定したバイトだけデータを送信
     * @param[in]    sd          ソケットディスクリプタ
     * @param[in]    size        データサイズ
     * @param[in]    data        送信データ
     */
    void tcpWriteBytes(int sd, int size, char *data, const struct timeval& deadline);
};

#endif // __Client_h__


----------------------------------------------------------------------------------------------------
ClientTcpIoWithC.cpp
----------------------------------------------------------------------------------------------------
/**
 * @file    ClientTcpIoWithC.cpp
 * @author  Copyright(C) Canon Inc. All right reserved.
 * @date    2014/01/10
 * @brief   TCP通信処理
 * @par     説明
 *  - TCP通信のクライアント側通信部分の機能を実現するクラス
 *  - (中間メッセージあり)
 */
#include <netdb.h>
#include <cstring>
#include "IF_ClientTcpIo.hpp"
#include "Client.hpp"
#include "TcpUtil.hpp"

// クライアントが並列で実行した場合、TCP情報は処理中に上書きされてしまうため、
// staticに保存する期間は最小に抑える必要がある。
S_TcpInfo* ClientTcpIoWithC::staticTcpInfo_;  ///< TCP情報
int        ClientTcpIoWithC::semId_ = -1;     ///< スレッド間受け渡しパラメータの保護用セマフォID

/**
 * @brief        コンストラクタ
 * @param[in]    tcpInfo     TCP情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    host        ホスト名
 * @param[in]    service     サービス名
 * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
 */
ClientTcpIoWithC::ClientTcpIoWithC(S_TcpInfo* tcpInfo, int fidNum, const char* host, const char* service)
	: UtilDetachThread("TcpSendThread", run), tcpInfo_(tcpInfo), host_(host), service_(service)
{
	connect_ = TCP_CONNECT; // connectなしコンストラクタは常時接続型扱い
	if (tcpInfo_->fid == NULL) {
		tcpInfo_->fidNum = fidNum;
		tcpInfo_->fid = new S_TcpFuncId [fidNum];
		for (int i = 0; i < tcpInfo_->fidNum; i++) {
			tcpInfo_->fid[i].funcId = i + 1;
			tcpInfo_->fid[i].sts    = E_FuncStsIdle;
		}
	}
}
ClientTcpIoWithC::ClientTcpIoWithC(S_TcpInfo* tcpInfo, int fidNum, const char* host, const char* service, int connect)
	: UtilDetachThread("TcpSendThread", run), tcpInfo_(tcpInfo), host_(host), service_(service), connect_(connect)
{
	if (tcpInfo_->fid == NULL) {
		tcpInfo_->fidNum = fidNum;
		tcpInfo_->fid = new S_TcpFuncId [fidNum];
		for (int i = 0; i < tcpInfo_->fidNum; i++) {
			tcpInfo_->fid[i].funcId = i + 1;
			tcpInfo_->fid[i].sts    = E_FuncStsIdle;
		}
	}
}

/**
 * @brief        デストラクタ
 */
ClientTcpIoWithC::~ClientTcpIoWithC()
{
}

/**
 * @brief        関数実行
 * @param[in]    id          関数ID
 * @param[in]    in          入力データ
 * @param[in]    size        入力データサイズ
 * @param[in]    timeout     タイムアウト時間 [usec]
 *
 * @retval       なし
 */
void ClientTcpIoWithC::exec(int id, char* in, int size, int timeout)
{
	int err = 0; ///< エラー戻り値

	log_printf(LV_DEBUG, "start\n");

	// 同一関数の連続実行は許可しない
	if ((getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsIdle) &&
			(getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsFinE)) {
		log_printf(LV_DEBUG, "same command can't exec.(%d)\n", id);
		throw ERR_TCPIO_FUNC_STS;
	}

	// 初回呼出はセマフォ初期化
	if (tcpInfo_->thread_semId == -1) {
		tcpInfo_->thread_semId = initSemaphore(Unlock);
	}
	if (tcpInfo_->c.semId == -1) {
		tcpInfo_->c.semId = initSemaphore(Unlock);
	}
	if (tcpInfo_->e.semId == -1) {
		tcpInfo_->e.semId = initSemaphore(Unlock);
	}

	// スレッド間受け渡しパラメータを保護するためのセマフォ
	if (semId_ == -1) {
		semId_ = initSemaphore(Unlock);
	}

	// セマフォ確保
	err = getSemaphore(tcpInfo_->thread_semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		postSemaphore(tcpInfo_->thread_semId);
		throw ERR_TCPIO_TIMEOUT_S;
	}
	err = getSemaphore(tcpInfo_->c.semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		throw ERR_TCPIO_TIMEOUT_S;
	}
	err = getSemaphore(tcpInfo_->e.semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		throw ERR_TCPIO_TIMEOUT_S;
	}

	// 初期化処理(IPアドレス情報、ポート番号情報を取得)
	Client obj;
	obj.TcpInit(host_, service_, &tcpInfo_->hostInfo, &tcpInfo_->portInfo);

	// 送信メッセージ領域確保
	// 関数実行毎に確保する
	if (tcpInfo_->s.in != NULL) {
		// waitE()で解放が望ましいが、マルチスレッドだと確保～send間で解放されるケースがある
		delete [] tcpInfo_->s.in;
		tcpInfo_->s.in = NULL;
	}
	int msgSize = SendFixMsgSize + size; // 固定部＋可変部
	tcpInfo_->s.in = new char [msgSize];
	if (tcpInfo_->s.in == NULL) {
		throw ERR_TCPIO_MALLOC;
	}
	memset(tcpInfo_->s.in, 0, sizeof(msgSize));
	log_printf(LV_DEBUG, "msgSize = %d(%d+%d)\n", msgSize, SendFixMsgSize, size);

	// 送信メッセージデータ格納
	// フォーマットは以下
	// 8Byte：送信ヘッダ(固定文字列)
	// 4Byte：関数ID
	// 8Byte：パリティ情報
	// 4Byte：データサイズ
	// 4Byte：中間メッセージ有無情報
	// 可変 ：送信パラメータ(バイナリ形式)
	// 8Byte：送信フッタ(固定文字列)
	struct timeval time;
	getTime(&time);
	int n = 7 + size / sizeof(int); // 送信フッタインデックス
	int* ptr = reinterpret_cast<int*>(tcpInfo_->s.in);
	memcpy(ptr, "SendHead", 2 * sizeof(int));   // 送信ヘッダ("SendHead")
	*(ptr + 2) = htonl(id);                     // 関数ID
	*(ptr + 3) = htonl((int)time.tv_sec);       // パリティ情報1(現在時刻sec部分)
	*(ptr + 4) = htonl((int)time.tv_usec);      // パリティ情報2(現在時刻usec部分)
	*(ptr + 5) = htonl(size);                   // データサイズ
	*(ptr + 6) = htonl(1);                      // 中間メッセージ有無情報(あり)
	memcpy(ptr + 7, in, size);                  // 送信パラメータ
	memcpy(ptr + n, "SendFoot", 2 * sizeof(int)); // 送信フッタ("SendFoot")
	tcpInfo_->s.size  = msgSize;                // 送信メッセージサイズ

	// データ保存
	setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsWaitC, id); // C完了待ち
	tcpInfo_->parity1 = (int)time.tv_sec;  // パリティ情報1
	tcpInfo_->parity2 = (int)time.tv_usec; // パリティ情報2
	tcpInfo_->connect = connect_;          // 接続タイプ(0:常時接続型/1:毎回接続型)
	log_printf(LV_DEBUG, "parity1=%d, parity2=%d\n", tcpInfo_->parity1, tcpInfo_->parity2);

	// TCP情報をstaticに保持する。
	// クライアントが並列で実行した場合、TCP情報は処理中に上書きされてしまうため、
	// スレッド処理開始直後にauto変数に退避する。
	// 受け渡しが終るまでセマフォで保護する。
	err = getSemaphore(semId_, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		throw ERR_TCPIO_TIMEOUT_S;
	}
	staticTcpInfo_ = tcpInfo_;

	// スレッド処理開始。run()を別スレッドで実行する。
	start();

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        スレッドで実行する処理
 *               スレッド処理が並列で動作することはない
 *
 * @retval       なし
 */
void ClientTcpIoWithC::run()
{
	const int RETRY_MAX = 30;        ///< メッセージ受信リトライ最大回数
	int       retry = 0;             ///< リトライループカウンタ
	char      header[HEADER_SIZE + 1]; ///< ヘッダ(固定文字列)
	int       funcId  = 0;           ///<  関数ID
	int       l_fid   = 0;           ///<  関数ID
	int       parity1 = 0;           ///<  パリティ情報1
	int       parity2 = 0;           ///<  パリティ情報2
	Client*    client = NULL;        ///< クライアントクラス
	bool      c_sem   = false;       ///< c.semId解放確認
	bool      e_sem   = false;       ///< e.semId解放確認
	bool      th_sem  = false;       ///< thread_semId解放確認

	log_printf(LV_DEBUG, "start\n");

	// static情報をauto変数に退避。
	// TCP情報が格納されている領域は、呼び出し元でstaticに確保している前提。
	S_TcpInfo* tcpInfo = staticTcpInfo_;
	l_fid = getFuncId(tcpInfo->fid, tcpInfo->fidNum);

	// 退避中はセマフォで保護
	postSemaphore(semId_);

	try {
		client = new Client(tcpInfo->hostInfo, tcpInfo->portInfo);

		// サーバ接続
		client->TcpConnect(&tcpInfo->sd, tcpInfo->connect);

		// サーバに送信メッセージ送信
		client->TcpWrite(tcpInfo->sd, tcpInfo->s.size, tcpInfo->s.in);

		// 中間メッセージ受信
		for (retry = 0; retry < RETRY_MAX; retry++) {

			// サーバから1メッセージ読み込み、送信メッセージに対する返信メッセージかをチェック
			memset(header, 0, sizeof(header));
			client->TcpRead(tcpInfo->sd, &tcpInfo->e.size, &tcpInfo->e.out, &tcpInfo->c.rtn,
							header, &funcId, &parity1, &parity2);
			if (checkMsg(MSG_TYPE_WITHC_C, l_fid, tcpInfo->parity1, tcpInfo->parity2,
						 header, funcId, parity1, parity2) == true) {
				break;
			}

			// 本スレッドがタイムアウト等で終了している場合は速やかに終了する
			if (getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle) {
				log_printf(LV_DEBUG, "this function must not continue.\n");
				throw ERR_TCPIO_RECEIVE;
			}

			// 規定回数メッセージを読み込んでも該当しない場合はエラーとする
			if (retry == RETRY_MAX - 1) {
				log_printf(LV_ERROR, "receive retry max error.(%d)\n", retry);
				throw ERR_TCPIO_RECEIVE;
			}
		}

		// セマフォ解放
		postSemaphore(tcpInfo->c.semId);
		c_sem  = true;

		// 返信メッセージ受信
		for (retry = 0; retry < RETRY_MAX; retry++) {

			// サーバから1メッセージ読み込み、送信メッセージに対する返信メッセージかをチェック
			memset(header, 0, sizeof(header));
			client->TcpRead(tcpInfo->sd, &tcpInfo->e.size, &tcpInfo->e.out, &tcpInfo->e.rtn,
							header, &funcId, &parity1, &parity2);
			if (checkMsg(MSG_TYPE_WITHC_E, l_fid, tcpInfo->parity1, tcpInfo->parity2,
						 header, funcId, parity1, parity2) == true) {
				break;
			}

			// 本スレッドがタイムアウト等で終了している場合は速やかに終了する
			if (getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle) {
				log_printf(LV_DEBUG, "this function must not continue.\n");
				throw ERR_TCPIO_RECEIVE;
			}

			// 規定回数メッセージを読み込んでも該当しない場合はエラーとする
			if (retry == RETRY_MAX - 1) {
				log_printf(LV_ERROR, "receive retry max error.(%d)\n", retry);
				throw ERR_TCPIO_RECEIVE;
			}
		}

		// 関数の戻り値チェック
		if (tcpInfo->e.rtn != 0) {
			log_printf(LV_ERROR, "exec function rtn error.(%d)\n", tcpInfo->e.rtn);
			throw tcpInfo->e.rtn;
		}

		// 関数終了
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid); // E完了

		// サーバ切断
		client->TcpClose(&tcpInfo->sd, tcpInfo->connect);

		// セマフォ解放
		postSemaphore(tcpInfo->e.semId);
		e_sem  = true;
		th_sem = true;

		delete client;
	}
	catch (int error) {
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid);
		client->TcpClose(&tcpInfo->sd, TCP_DISCONNECT);
		tcpInfo->c.rtn = error;
		tcpInfo->e.rtn = error;
		if (!c_sem)  postSemaphore(tcpInfo->c.semId);
		if (!e_sem)  postSemaphore(tcpInfo->e.semId);
		if (!th_sem) postSemaphore(tcpInfo->thread_semId);
		delete client;
		throw error;
	}
	catch (...) {
		log_printf(LV_ERROR, "error.\n");
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid);
		client->TcpClose(&tcpInfo->sd, TCP_DISCONNECT);
		tcpInfo->c.rtn = -1;
		tcpInfo->e.rtn = -1;
		if (!c_sem)  postSemaphore(tcpInfo->c.semId);
		if (!e_sem)  postSemaphore(tcpInfo->e.semId);
		if (!th_sem) postSemaphore(tcpInfo->thread_semId);
		delete client;
		throw - 1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        中間メッセージ受信
 * @param[in]    id          関数ID
 * @param[in]    timeout     タイムアウト時間 [usec]
 *
 * @retval       なし
 */
int ClientTcpIoWithC::waitC(int id, int timeout)
{
	int err = 0; ///< エラー戻り値

	log_printf(LV_DEBUG, "start\n");

	// 関数実行状態チェック
	// C()完了待ちでないとC()は実行できない
	if ((getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsWaitC) &&
			(getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsFinE)) {
		log_printf(LV_DEBUG, "funcStatus is wrong.(%d)\n", getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id));
		throw ERR_TCPIO_FUNC_STS;
	}

	// セマフォ確保
	err = getSemaphore(tcpInfo_->c.semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsIdle, id);
		postSemaphore(tcpInfo_->thread_semId);
		throw ERR_TCPIO_TIMEOUT_C;
	}

	// セマフォ解放
	postSemaphore(tcpInfo_->c.semId);

	// 関数実行状態変更
	setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsWaitE, id);

	log_printf(LV_DEBUG, "end\n");

	return tcpInfo_->c.rtn;
}

/**
 * @brief        終了メッセージ受信
 * @param[in]    id          関数ID
 * @param[out]   out         出力データ
 * @param[in]    size        出力データサイズ
 * @param[in]    timeout     タイムアウト時間 [usec]
 *
 * @retval       なし
 */
void ClientTcpIoWithC::waitE(int id, char* out, int size, int timeout)
{
	int err = 0; ///< エラー戻り値

	log_printf(LV_DEBUG, "start\n");

	// 関数実行状態チェック
	// E()完了待ちでないとE()は実行できない
	if ((getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsWaitE) &&
			(getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsFinE)) {
		// ただし、同一関数の場合はC()が実行されていなかった場合はwaitC()を呼び処理を継続する
		if (getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) == E_FuncStsWaitC) {
			log_printf(LV_DEBUG, "not call waitC\n");
			waitC(id, timeout);
		}
		else {
			log_printf(LV_DEBUG, "funcStatus is wrong.(%d)\n", getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id));
			throw ERR_TCPIO_FUNC_STS;
		}
	}

	// セマフォ確保
	err = getSemaphore(tcpInfo_->e.semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsIdle, id);
		postSemaphore(tcpInfo_->thread_semId);
		throw ERR_TCPIO_TIMEOUT_E;
	}

	// セマフォ解放
	postSemaphore(tcpInfo_->e.semId);
	postSemaphore(tcpInfo_->thread_semId);

	// 関数実行状態変更
	setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsIdle, id);

	// 出力データ受け取り
	if (tcpInfo_->e.out != NULL) {
		memcpy(out, tcpInfo_->e.out, size);
	}

	// 関数戻り値チェック
	if (tcpInfo_->e.rtn != 0) {
		throw tcpInfo_->e.rtn;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        メッセージチェック
 * @param[in]    msgType     メッセージタイプ
 * @param[in]    inFuncId    入力関数ID
 * @param[in]    inParity1   入力パリティ情報1
 * @param[in]    inParity2   入力パリティ情報2
 * @param[in]    outHeader   出力ヘッダ
 * @param[in]    outFuncId   出力関数ID
 * @param[in]    outParity1  出力パリティ情報1
 * @param[in]    outParity2  出力パリティ情報2
 *
 * @retval       true        メッセージ一致
 * @retval       false       メッセージ不一致
 */
bool ClientTcpIoWithC::checkMsg(E_MsgTypeWithC msgType, int inFuncId, int inParity1, int inParity2,
								char* outHeader, int outFuncId, int outParity1, int outParity2)
{
	// ヘッダが一致しているか？
	switch (msgType) {
	case MSG_TYPE_WITHC_S:
		if (strcmp(outHeader, "SendHead") != 0) {
			log_printf(LV_DEBUG, "header no match(SendHead vs %s)\n", outHeader);
			return false;
		}
		break;
	case MSG_TYPE_WITHC_C:
		if (strcmp(outHeader, "InfmHead") != 0) {
			log_printf(LV_DEBUG, "header no match(InfmHead vs %s)\n", outHeader);
			return false;
		}
		break;
	case MSG_TYPE_WITHC_E:
		if (strcmp(outHeader, "RecvHead") != 0) {
			log_printf(LV_DEBUG, "header no match(RecvHead vs %s)\n", outHeader);
			return false;
		}
		break;
	}

	// 関数IDが一致しているか？
	if (inFuncId != outFuncId) {
		log_printf(LV_DEBUG, "funcId no match(%d vs %d)\n", inFuncId, outFuncId);
		return false;
	}

	// パリティ情報1が一致しているか？
	if (inParity1 != outParity1) {
		log_printf(LV_DEBUG, "parity1 no match(%d vs %d)\n", inParity1, outParity1);
		return false;
	}

	// パリティ情報2が一致しているか？
	if (inParity2 != outParity2) {
		log_printf(LV_DEBUG, "parity2 no match(%d vs %d)\n", inParity2, outParity2);
		return false;
	}

	return true;
}

/**
 * @brief        関数実行状態取得
 * @param[in]    fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    funcId      関数ID
 *
 * @retval       関数実行状態(0:Idle状態/1:C完了待ち/2:E完了待ち/3:E完了)
 */
E_FuncSts ClientTcpIoWithC::getFuncIdSts(S_TcpFuncId* fid, int fidNum, int funcId)
{
	E_FuncSts sts = E_FuncStsIdle;

	for (int i = 0; i < fidNum; i++) {
		if (funcId == fid[i].funcId) {
			sts = fid[i].sts;
		}
	}
	return sts;
}

/**
 * @brief        関数実行状態設定
 * @param[inout] fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    sts         関数実行状態(0:Idle状態/1:C完了待ち/2:E完了待ち/3:E完了)
 * @param[in]    funcId      関数ID
 *
 * @retval       なし
 */
void ClientTcpIoWithC::setFuncIdSts(S_TcpFuncId* fid, int fidNum, E_FuncSts sts, int funcId)
{
	for (int i = 0; i < fidNum; i++) {
		if (funcId == fid[i].funcId) {
			fid[i].sts = sts;
			log_printf(LV_DEBUG, "funcStatus(fid:%d) = %d\n", i + 1, fid[i].sts);
		}
	}
	return;
}

/**
 * @brief        関数ID取得
 * @param[in]    fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 *
 * @retval       関数ID
 */
int ClientTcpIoWithC::getFuncId(S_TcpFuncId* fid, int fidNum)
{
	int funcId = 0;

	// 実行中の関数は同時には1つしかないことを前提として
	// 現在実行中の関数IDを取得する
	for (int i = 0; i < fidNum; i++) {
		if ((fid[i].sts != E_FuncStsIdle) &&
				(fid[i].sts != E_FuncStsFinE)) {
			funcId = fid[i].funcId;
		}
	}
	return funcId;
}


----------------------------------------------------------------------------------------------------
/**
 * @file    ClientTcpIoWithC.cpp
 * @author  Copyright(C) Canon Inc. All right reserved.
 * @date    2014/01/10
 * @brief   TCP通信処理
 * @par     説明
 *  - TCP通信のクライアント側通信部分の機能を実現するクラス
 *  - (中間メッセージあり)
 */
#include <netdb.h>
#include <cstring>
#include "IF_ClientTcpIo.hpp"
#include "Client.hpp"
#include "TcpUtil.hpp"

// クライアントが並列で実行した場合、TCP情報は処理中に上書きされてしまうため、
// staticに保存する期間は最小に抑える必要がある。
S_TcpInfo* ClientTcpIoWithC::staticTcpInfo_;  ///< TCP情報
int        ClientTcpIoWithC::semId_ = -1;     ///< スレッド間受け渡しパラメータの保護用セマフォID

/**
 * @brief        コンストラクタ
 * @param[in]    tcpInfo     TCP情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    host        ホスト名
 * @param[in]    service     サービス名
 * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
 */
ClientTcpIoWithC::ClientTcpIoWithC(S_TcpInfo* tcpInfo, int fidNum, const char* host, const char* service)
	: UtilDetachThread("TcpSendThread", run), tcpInfo_(tcpInfo), host_(host), service_(service)
{
	connect_ = TCP_CONNECT; // connectなしコンストラクタは常時接続型扱い
	if (tcpInfo_->fid == NULL) {
		tcpInfo_->fidNum = fidNum;
		tcpInfo_->fid = new S_TcpFuncId [fidNum];
		for (int i = 0; i < tcpInfo_->fidNum; i++) {
			tcpInfo_->fid[i].funcId = i + 1;
			tcpInfo_->fid[i].sts    = E_FuncStsIdle;
		}
	}
}
ClientTcpIoWithC::ClientTcpIoWithC(S_TcpInfo* tcpInfo, int fidNum, const char* host, const char* service, int connect)
	: UtilDetachThread("TcpSendThread", run), tcpInfo_(tcpInfo), host_(host), service_(service), connect_(connect)
{
	if (tcpInfo_->fid == NULL) {
		tcpInfo_->fidNum = fidNum;
		tcpInfo_->fid = new S_TcpFuncId [fidNum];
		for (int i = 0; i < tcpInfo_->fidNum; i++) {
			tcpInfo_->fid[i].funcId = i + 1;
			tcpInfo_->fid[i].sts    = E_FuncStsIdle;
		}
	}
}

/**
 * @brief        デストラクタ
 */
ClientTcpIoWithC::~ClientTcpIoWithC()
{
}

/**
 * @brief        関数実行
 * @param[in]    id          関数ID
 * @param[in]    in          入力データ
 * @param[in]    size        入力データサイズ
 * @param[in]    timeout     タイムアウト時間 [usec]
 *
 * @retval       なし
 */
void ClientTcpIoWithC::exec(int id, char* in, int size, int timeout)
{
	int err = 0; ///< エラー戻り値

	log_printf(LV_DEBUG, "start\n");

	// 同一関数の連続実行は許可しない
	if ((getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsIdle) &&
			(getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsFinE)) {
		log_printf(LV_DEBUG, "same command can't exec.(%d)\n", id);
		throw ERR_TCPIO_FUNC_STS;
	}

	// 初回呼出はセマフォ初期化
	if (tcpInfo_->thread_semId == -1) {
		tcpInfo_->thread_semId = initSemaphore(Unlock);
	}
	if (tcpInfo_->c.semId == -1) {
		tcpInfo_->c.semId = initSemaphore(Unlock);
	}
	if (tcpInfo_->e.semId == -1) {
		tcpInfo_->e.semId = initSemaphore(Unlock);
	}

	// スレッド間受け渡しパラメータを保護するためのセマフォ
	if (semId_ == -1) {
		semId_ = initSemaphore(Unlock);
	}

	// セマフォ確保
	err = getSemaphore(tcpInfo_->thread_semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		postSemaphore(tcpInfo_->thread_semId);
		throw ERR_TCPIO_TIMEOUT_S;
	}
	err = getSemaphore(tcpInfo_->c.semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		throw ERR_TCPIO_TIMEOUT_S;
	}
	err = getSemaphore(tcpInfo_->e.semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		throw ERR_TCPIO_TIMEOUT_S;
	}

	// 初期化処理(IPアドレス情報、ポート番号情報を取得)
	Client obj;
	obj.TcpInit(host_, service_, &tcpInfo_->hostInfo, &tcpInfo_->portInfo);

	// 送信メッセージ領域確保
	// 関数実行毎に確保する
	if (tcpInfo_->s.in != NULL) {
		// waitE()で解放が望ましいが、マルチスレッドだと確保〜send間で解放されるケースがある
		delete [] tcpInfo_->s.in;
		tcpInfo_->s.in = NULL;
	}
	int msgSize = SendFixMsgSize + size; // 固定部＋可変部
	tcpInfo_->s.in = new char [msgSize];
	if (tcpInfo_->s.in == NULL) {
		throw ERR_TCPIO_MALLOC;
	}
	memset(tcpInfo_->s.in, 0, sizeof(msgSize));
	log_printf(LV_DEBUG, "msgSize = %d(%d+%d)\n", msgSize, SendFixMsgSize, size);

	// 送信メッセージデータ格納
	// フォーマットは以下
	// 8Byte：送信ヘッダ(固定文字列)
	// 4Byte：関数ID
	// 8Byte：パリティ情報
	// 4Byte：データサイズ
	// 4Byte：中間メッセージ有無情報
	// 可変 ：送信パラメータ(バイナリ形式)
	// 8Byte：送信フッタ(固定文字列)
	struct timeval time;
	getTime(&time);
	int n = 7 + size / sizeof(int); // 送信フッタインデックス
	int* ptr = reinterpret_cast<int*>(tcpInfo_->s.in);
	memcpy(ptr, "SendHead", 2 * sizeof(int));   // 送信ヘッダ("SendHead")
	*(ptr + 2) = htonl(id);                     // 関数ID
	*(ptr + 3) = htonl((int)time.tv_sec);       // パリティ情報1(現在時刻sec部分)
	*(ptr + 4) = htonl((int)time.tv_usec);      // パリティ情報2(現在時刻usec部分)
	*(ptr + 5) = htonl(size);                   // データサイズ
	*(ptr + 6) = htonl(1);                      // 中間メッセージ有無情報(あり)
	memcpy(ptr + 7, in, size);                  // 送信パラメータ
	memcpy(ptr + n, "SendFoot", 2 * sizeof(int)); // 送信フッタ("SendFoot")
	tcpInfo_->s.size  = msgSize;                // 送信メッセージサイズ

	// データ保存
	setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsWaitC, id); // C完了待ち
	tcpInfo_->parity1 = (int)time.tv_sec;  // パリティ情報1
	tcpInfo_->parity2 = (int)time.tv_usec; // パリティ情報2
	tcpInfo_->connect = connect_;          // 接続タイプ(0:常時接続型/1:毎回接続型)
	log_printf(LV_DEBUG, "parity1=%d, parity2=%d\n", tcpInfo_->parity1, tcpInfo_->parity2);

	// TCP情報をstaticに保持する。
	// クライアントが並列で実行した場合、TCP情報は処理中に上書きされてしまうため、
	// スレッド処理開始直後にauto変数に退避する。
	// 受け渡しが終るまでセマフォで保護する。
	err = getSemaphore(semId_, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		throw ERR_TCPIO_TIMEOUT_S;
	}
	staticTcpInfo_ = tcpInfo_;

	// スレッド処理開始。run()を別スレッドで実行する。
	start();

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        スレッドで実行する処理
 *               スレッド処理が並列で動作することはない
 *
 * @retval       なし
 */
void ClientTcpIoWithC::run()
{
	const int RETRY_MAX = 30;        ///< メッセージ受信リトライ最大回数
	int       retry = 0;             ///< リトライループカウンタ
	char      header[HEADER_SIZE + 1]; ///< ヘッダ(固定文字列)
	int       funcId  = 0;           ///<  関数ID
	int       l_fid   = 0;           ///<  関数ID
	int       parity1 = 0;           ///<  パリティ情報1
	int       parity2 = 0;           ///<  パリティ情報2
	Client*    client = NULL;        ///< クライアントクラス
	bool      c_sem   = false;       ///< c.semId解放確認
	bool      e_sem   = false;       ///< e.semId解放確認
	bool      th_sem  = false;       ///< thread_semId解放確認

	log_printf(LV_DEBUG, "start\n");

	// static情報をauto変数に退避。
	// TCP情報が格納されている領域は、呼び出し元でstaticに確保している前提。
	S_TcpInfo* tcpInfo = staticTcpInfo_;
	l_fid = getFuncId(tcpInfo->fid, tcpInfo->fidNum);

	// 退避中はセマフォで保護
	postSemaphore(semId_);

	try {
		client = new Client(tcpInfo->hostInfo, tcpInfo->portInfo);

		// サーバ接続
		client->TcpConnect(&tcpInfo->sd, tcpInfo->connect);

		// 目的: 接続待ち中に呼び出し元がタイムアウトした場合、不要な送受信を開始しない。
		// 処理: 都度接続モードで関数状態がIdleなら、共通の例外後始末へ進む。
		if ((tcpInfo->connect == TCP_DISCONNECT) &&
				// 呼び出し側が待ちを打ち切った状態か確認する。
				(getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle)) {
			// 後続の通信を中止し、例外処理でFDを閉じる。
			throw ERR_TCPIO_FUNC_STS;
		}

		// サーバに送信メッセージ送信
		client->TcpWrite(tcpInfo->sd, tcpInfo->s.size, tcpInfo->s.in);

		// 目的: 送信中に呼び出し元がタイムアウトした場合、応答受信待ちへ進まない。
		// 処理: 都度接続モードでIdleを検出し、共通の例外後始末でソケットを閉じる。
		if ((tcpInfo->connect == TCP_DISCONNECT) &&
				// 送信中に呼び出し側がタイムアウト状態へ変更したか確認する。
				(getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle)) {
			// 通信を中止し、後続のクローズ処理へ移す。
			throw ERR_TCPIO_FUNC_STS;
		}

		// 中間メッセージ受信
		for (retry = 0; retry < RETRY_MAX; retry++) {

			// サーバから1メッセージ読み込み、送信メッセージに対する返信メッセージかをチェック
			memset(header, 0, sizeof(header));
			client->TcpRead(tcpInfo->sd, &tcpInfo->e.size, &tcpInfo->e.out, &tcpInfo->c.rtn,
							header, &funcId, &parity1, &parity2);
			if (checkMsg(MSG_TYPE_WITHC_C, l_fid, tcpInfo->parity1, tcpInfo->parity2,
						 header, funcId, parity1, parity2) == true) {
				break;
			}

			// 本スレッドがタイムアウト等で終了している場合は速やかに終了する
			if (getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle) {
				log_printf(LV_DEBUG, "this function must not continue.\n");
				throw ERR_TCPIO_RECEIVE;
			}

			// 規定回数メッセージを読み込んでも該当しない場合はエラーとする
			if (retry == RETRY_MAX - 1) {
				log_printf(LV_ERROR, "receive retry max error.(%d)\n", retry);
				throw ERR_TCPIO_RECEIVE;
			}
		}

		// セマフォ解放
		postSemaphore(tcpInfo->c.semId);
		c_sem  = true;

		// 返信メッセージ受信
		for (retry = 0; retry < RETRY_MAX; retry++) {

			// サーバから1メッセージ読み込み、送信メッセージに対する返信メッセージかをチェック
			memset(header, 0, sizeof(header));
			client->TcpRead(tcpInfo->sd, &tcpInfo->e.size, &tcpInfo->e.out, &tcpInfo->e.rtn,
							header, &funcId, &parity1, &parity2);
			if (checkMsg(MSG_TYPE_WITHC_E, l_fid, tcpInfo->parity1, tcpInfo->parity2,
						 header, funcId, parity1, parity2) == true) {
				break;
			}

			// 本スレッドがタイムアウト等で終了している場合は速やかに終了する
			if (getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle) {
				log_printf(LV_DEBUG, "this function must not continue.\n");
				throw ERR_TCPIO_RECEIVE;
			}

			// 規定回数メッセージを読み込んでも該当しない場合はエラーとする
			if (retry == RETRY_MAX - 1) {
				log_printf(LV_ERROR, "receive retry max error.(%d)\n", retry);
				throw ERR_TCPIO_RECEIVE;
			}
		}

		// 関数の戻り値チェック
		if (tcpInfo->e.rtn != 0) {
			log_printf(LV_ERROR, "exec function rtn error.(%d)\n", tcpInfo->e.rtn);
			throw tcpInfo->e.rtn;
		}

		// 関数終了
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid); // E完了

		// サーバ切断
		client->TcpClose(&tcpInfo->sd, tcpInfo->connect);

		// セマフォ解放
		postSemaphore(tcpInfo->e.semId);
		e_sem  = true;
		th_sem = true;

		delete client;
	}
	catch (int error) {
		// 整数エラーを記録し、C/Eセマフォとソケットの後始末を行う。
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		// 目的: タイムアウト側とワーカー側のFDリーク・セマフォ二重解放を防ぐ。
		// IdleならwaitCまたはwaitEがすでにタイムアウト処理を開始している。
		bool callerTimedOut = (getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle);
		// 呼び出し元へエラー完了を伝えるため、関数状態を完了状態に更新する。
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid);
		// ClientとFDが有効なときだけ、都度接続したソケットを閉じる。
		if ((client != NULL) && (tcpInfo->sd >= 0)) {
			// TCP_DISCONNECTでshutdown/closeし、共有FDを無効化する。
			client->TcpClose(&tcpInfo->sd, TCP_DISCONNECT);
		}
		// C完了待ちへ今回のエラーを返すため、戻り値領域へ保存する。
		tcpInfo->c.rtn = error;
		// E完了待ちへも同じエラーを返せるよう、戻り値領域へ保存する。
		tcpInfo->e.rtn = error;
		// C完了セマフォをまだ通知していない場合だけ、待ち側へ通知する。
		if (!c_sem)  postSemaphore(tcpInfo->c.semId);
		// E完了セマフォをまだ通知していない場合だけ、待ち側へ通知する。
		if (!e_sem)  postSemaphore(tcpInfo->e.semId);
		// タイムアウト側が解放済みでない場合だけthreadセマフォを解放する。
		if (!th_sem && !callerTimedOut) postSemaphore(tcpInfo->thread_semId);
		// ワーカースレッドが作成したClientオブジェクトを破棄する。
		delete client;
		// 元の整数エラーを維持して呼び出し元へ返す。
		throw error;
	}
	catch (...) {
		// 整数以外の例外も記録し、同じ資源解放経路で処理する。
		log_printf(LV_ERROR, "error.\n");
		// 目的: 想定外の例外でもFDリークとセマフォ二重解放を防ぐ。
		// IdleならwaitCまたはwaitEがすでにタイムアウト処理を開始している。
		bool callerTimedOut = (getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle);
		// 呼び出し元へ終了を伝えるため、関数状態を完了状態に更新する。
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid);
		// ClientとFDが有効な場合だけ、都度接続したソケットを閉じる。
		if ((client != NULL) && (tcpInfo->sd >= 0)) {
			// TCP_DISCONNECTでソケットを閉じ、共有FDを無効化する。
			client->TcpClose(&tcpInfo->sd, TCP_DISCONNECT);
		}
		// C完了待ちへ汎用エラーを返すため、戻り値領域へ保存する。
		tcpInfo->c.rtn = -1;
		// E完了待ちへ汎用エラーを返すため、戻り値領域へ保存する。
		tcpInfo->e.rtn = -1;
		// C完了セマフォが未通知の場合だけ、待ち側へ通知する。
		if (!c_sem)  postSemaphore(tcpInfo->c.semId);
		// E完了セマフォが未通知の場合だけ、待ち側へ通知する。
		if (!e_sem)  postSemaphore(tcpInfo->e.semId);
		// タイムアウト側が解放済みでない場合だけthreadセマフォを解放する。
		if (!th_sem && !callerTimedOut) postSemaphore(tcpInfo->thread_semId);
		// ワーカースレッドが作成したClientオブジェクトを破棄する。
		delete client;
		// 想定外の例外を汎用エラー値に置き換えて呼び出し元へ返す。
		throw - 1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        中間メッセージ受信
 * @param[in]    id          関数ID
 * @param[in]    timeout     タイムアウト時間 [usec]
 *
 * @retval       なし
 */
int ClientTcpIoWithC::waitC(int id, int timeout)
{
	int err = 0; ///< エラー戻り値

	log_printf(LV_DEBUG, "start\n");

	// 関数実行状態チェック
	// C()完了待ちでないとC()は実行できない
	if ((getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsWaitC) &&
			(getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsFinE)) {
		log_printf(LV_DEBUG, "funcStatus is wrong.(%d)\n", getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id));
		throw ERR_TCPIO_FUNC_STS;
	}

	// セマフォ確保
	err = getSemaphore(tcpInfo_->c.semId, timeout);
	if (err == -1) {
		// 指定時間内に中間メッセージ完了セマフォを取得できなかったことを記録する。
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		// ワーカースレッドへタイムアウトを知らせ、受信処理を終了させる。
		setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsIdle, id);
		// 目的: C待ちタイムアウト時にrecvを解除し、ワーカーに都度接続FDを閉じさせる。
		// 常時接続モードのソケットは維持し、都度接続モードかつFD有効の場合だけ切断する。
		if ((connect_ == TCP_DISCONNECT) && (tcpInfo_->sd >= 0)) {
			// 送受信を停止し、ブロッキング中のrecvを解除する。
			err = shutdown(tcpInfo_->sd, SHUT_RDWR);
			// shutdown失敗時もCタイムアウトの通知は続けるため、原因のみ記録する。
			if (err < 0) {
				// FD番号とOSエラーを記録し、ソケット停止の失敗を明示する。
				log_printf(LV_ERROR, "shutdown error.(err = %d, errno = %d)\n", err, errno);
			}
		}
		// C待ち呼び出し側が保持するthreadセマフォを解放する。
		postSemaphore(tcpInfo_->thread_semId);
		// C完了待ちのタイムアウトを呼び出し元へ通知する。
		throw ERR_TCPIO_TIMEOUT_C;
	}

	// セマフォ解放
	postSemaphore(tcpInfo_->c.semId);

	// 関数実行状態変更
	setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsWaitE, id);

	log_printf(LV_DEBUG, "end\n");

	return tcpInfo_->c.rtn;
}

/**
 * @brief        終了メッセージ受信
 * @param[in]    id          関数ID
 * @param[out]   out         出力データ
 * @param[in]    size        出力データサイズ
 * @param[in]    timeout     タイムアウト時間 [usec]
 *
 * @retval       なし
 */
void ClientTcpIoWithC::waitE(int id, char* out, int size, int timeout)
{
	int err = 0; ///< エラー戻り値

	log_printf(LV_DEBUG, "start\n");

	// 関数実行状態チェック
	// E()完了待ちでないとE()は実行できない
	if ((getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsWaitE) &&
			(getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsFinE)) {
		// ただし、同一関数の場合はC()が実行されていなかった場合はwaitC()を呼び処理を継続する
		if (getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) == E_FuncStsWaitC) {
			log_printf(LV_DEBUG, "not call waitC\n");
			waitC(id, timeout);
		}
		else {
			log_printf(LV_DEBUG, "funcStatus is wrong.(%d)\n", getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id));
			throw ERR_TCPIO_FUNC_STS;
		}
	}

	// セマフォ確保
	err = getSemaphore(tcpInfo_->e.semId, timeout);
	if (err == -1) {
		// 指定時間内に終了メッセージ完了セマフォを取得できなかったことを記録する。
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		// ワーカースレッドへタイムアウトを知らせ、受信処理を終了させる。
		setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsIdle, id);
		// 目的: E待ちタイムアウト時にrecvを解除し、ワーカーに都度接続FDを閉じさせる。
		// 常時接続モードのソケットは維持し、都度接続モードかつFD有効の場合だけ切断する。
		if ((connect_ == TCP_DISCONNECT) && (tcpInfo_->sd >= 0)) {
			// 送受信を停止し、ブロッキング中のrecvを解除する。
			err = shutdown(tcpInfo_->sd, SHUT_RDWR);
			// shutdown失敗時もEタイムアウトの通知は続けるため、原因のみ記録する。
			if (err < 0) {
				// FD番号とOSエラーを記録し、ソケット停止の失敗を明示する。
				log_printf(LV_ERROR, "shutdown error.(err = %d, errno = %d)\n", err, errno);
			}
		}
		// E待ち呼び出し側が保持するthreadセマフォを解放する。
		postSemaphore(tcpInfo_->thread_semId);
		// E完了待ちのタイムアウトを呼び出し元へ通知する。
		throw ERR_TCPIO_TIMEOUT_E;
	}

	// セマフォ解放
	postSemaphore(tcpInfo_->e.semId);
	postSemaphore(tcpInfo_->thread_semId);

	// 関数実行状態変更
	setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsIdle, id);

	// 出力データ受け取り
	if (tcpInfo_->e.out != NULL) {
		memcpy(out, tcpInfo_->e.out, size);
	}

	// 関数戻り値チェック
	if (tcpInfo_->e.rtn != 0) {
		throw tcpInfo_->e.rtn;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        メッセージチェック
 * @param[in]    msgType     メッセージタイプ
 * @param[in]    inFuncId    入力関数ID
 * @param[in]    inParity1   入力パリティ情報1
 * @param[in]    inParity2   入力パリティ情報2
 * @param[in]    outHeader   出力ヘッダ
 * @param[in]    outFuncId   出力関数ID
 * @param[in]    outParity1  出力パリティ情報1
 * @param[in]    outParity2  出力パリティ情報2
 *
 * @retval       true        メッセージ一致
 * @retval       false       メッセージ不一致
 */
bool ClientTcpIoWithC::checkMsg(E_MsgTypeWithC msgType, int inFuncId, int inParity1, int inParity2,
								char* outHeader, int outFuncId, int outParity1, int outParity2)
{
	// ヘッダが一致しているか？
	switch (msgType) {
	case MSG_TYPE_WITHC_S:
		if (strcmp(outHeader, "SendHead") != 0) {
			log_printf(LV_DEBUG, "header no match(SendHead vs %s)\n", outHeader);
			return false;
		}
		break;
	case MSG_TYPE_WITHC_C:
		if (strcmp(outHeader, "InfmHead") != 0) {
			log_printf(LV_DEBUG, "header no match(InfmHead vs %s)\n", outHeader);
			return false;
		}
		break;
	case MSG_TYPE_WITHC_E:
		if (strcmp(outHeader, "RecvHead") != 0) {
			log_printf(LV_DEBUG, "header no match(RecvHead vs %s)\n", outHeader);
			return false;
		}
		break;
	}

	// 関数IDが一致しているか？
	if (inFuncId != outFuncId) {
		log_printf(LV_DEBUG, "funcId no match(%d vs %d)\n", inFuncId, outFuncId);
		return false;
	}

	// パリティ情報1が一致しているか？
	if (inParity1 != outParity1) {
		log_printf(LV_DEBUG, "parity1 no match(%d vs %d)\n", inParity1, outParity1);
		return false;
	}

	// パリティ情報2が一致しているか？
	if (inParity2 != outParity2) {
		log_printf(LV_DEBUG, "parity2 no match(%d vs %d)\n", inParity2, outParity2);
		return false;
	}

	return true;
}

/**
 * @brief        関数実行状態取得
 * @param[in]    fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    funcId      関数ID
 *
 * @retval       関数実行状態(0:Idle状態/1:C完了待ち/2:E完了待ち/3:E完了)
 */
E_FuncSts ClientTcpIoWithC::getFuncIdSts(S_TcpFuncId* fid, int fidNum, int funcId)
{
	E_FuncSts sts = E_FuncStsIdle;

	for (int i = 0; i < fidNum; i++) {
		if (funcId == fid[i].funcId) {
			sts = fid[i].sts;
		}
	}
	return sts;
}

/**
 * @brief        関数実行状態設定
 * @param[inout] fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    sts         関数実行状態(0:Idle状態/1:C完了待ち/2:E完了待ち/3:E完了)
 * @param[in]    funcId      関数ID
 *
 * @retval       なし
 */
void ClientTcpIoWithC::setFuncIdSts(S_TcpFuncId* fid, int fidNum, E_FuncSts sts, int funcId)
{
	for (int i = 0; i < fidNum; i++) {
		if (funcId == fid[i].funcId) {
			fid[i].sts = sts;
			log_printf(LV_DEBUG, "funcStatus(fid:%d) = %d\n", i + 1, fid[i].sts);
		}
	}
	return;
}

/**
 * @brief        関数ID取得
 * @param[in]    fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 *
 * @retval       関数ID
 */
int ClientTcpIoWithC::getFuncId(S_TcpFuncId* fid, int fidNum)
{
	int funcId = 0;

	// 実行中の関数は同時には1つしかないことを前提として
	// 現在実行中の関数IDを取得する
	for (int i = 0; i < fidNum; i++) {
		if ((fid[i].sts != E_FuncStsIdle) &&
				(fid[i].sts != E_FuncStsFinE)) {
			funcId = fid[i].funcId;
		}
	}
	return funcId;
}


----------------------------------------------------------------------------------------------------
ClientTcpIo.cpp
----------------------------------------------------------------------------------------------------
/**
 * @file    ClientTcpIo.cpp
 * @author  Copyright(C) Canon Inc. All right reserved.
 * @date    2014/01/10
 * @brief   TCP通信処理
 * @par     説明
 *  - TCP通信のクライアント側通信部分の機能を実現するクラス
 *  - (中間メッセージなし)
 */
#include <netdb.h>
#include <cstring>
#include "IF_ClientTcpIo.hpp"
#include "Client.hpp"
#include "TcpUtil.hpp"

// クライアントが並列で実行した場合、TCP情報は処理中に上書きされてしまうため、
// staticに保存する期間は最小に抑える必要がある。
S_TcpInfo* ClientTcpIo::staticTcpInfo_;  ///< TCP情報
int        ClientTcpIo::semId_ = -1;      ///< スレッド間受け渡しパラメータの保護用セマフォID

/**
 * @brief        コンストラクタ
 * @param[in]    tcpInfo     TCP情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    host        ホスト名
 * @param[in]    service     サービス名
 * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
 */
ClientTcpIo::ClientTcpIo(S_TcpInfo* tcpInfo, int fidNum, const char* host, const char* service)
	: UtilDetachThread("TcpSendThread", run), tcpInfo_(tcpInfo), host_(host), service_(service)
{
	connect_ = TCP_CONNECT; // connectなしコンストラクタは常時接続型扱い
	if (tcpInfo_->fid == NULL) {
		tcpInfo_->fidNum = fidNum;
		tcpInfo_->fid = new S_TcpFuncId [fidNum];
		for (int i = 0; i < tcpInfo_->fidNum; i++) {
			tcpInfo_->fid[i].funcId = i + 1;
			tcpInfo_->fid[i].sts    = E_FuncStsIdle;
		}
	}
}
ClientTcpIo::ClientTcpIo(S_TcpInfo* tcpInfo, int fidNum, const char* host, const char* service, int connect)
	: UtilDetachThread("TcpSendThread", run), tcpInfo_(tcpInfo), host_(host), service_(service), connect_(connect)
{
	if (tcpInfo_->fid == NULL) {
		tcpInfo_->fidNum = fidNum;
		tcpInfo_->fid = new S_TcpFuncId [fidNum];
		for (int i = 0; i < tcpInfo_->fidNum; i++) {
			tcpInfo_->fid[i].funcId = i + 1;
			tcpInfo_->fid[i].sts    = E_FuncStsIdle;
		}
	}
}

/**
 * @brief        デストラクタ
 */
ClientTcpIo::~ClientTcpIo()
{
}

/**
 * @brief        関数実行
 * @param[in]    id          関数ID
 * @param[in]    in          入力データ
 * @param[in]    size        入力データサイズ
 * @param[in]    timeout     タイムアウト時間 [usec]
 *
 * @retval       なし
 */
void ClientTcpIo::exec(int id, char* in, int size, int timeout)
{
	int err = 0; ///< エラー戻り値

	log_printf(LV_DEBUG, "start\n");

	// 同一関数の連続実行は許可しない
	if ((getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsIdle) &&
			(getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsFinE)) {
		log_printf(LV_DEBUG, "same command can't exec.(%d)\n", id);
		throw ERR_TCPIO_FUNC_STS;
	}

	// 初回呼出はセマフォ初期化
	if (tcpInfo_->thread_semId == -1) {
		tcpInfo_->thread_semId = initSemaphore(Unlock);
	}
	if (tcpInfo_->e.semId == -1) {
		tcpInfo_->e.semId = initSemaphore(Unlock);
	}

	// スレッド間受け渡しパラメータを保護するためのセマフォ
	if (semId_ == -1) {
		semId_ = initSemaphore(Unlock);
	}

	// セマフォ確保
	err = getSemaphore(tcpInfo_->thread_semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		postSemaphore(tcpInfo_->thread_semId);
		throw ERR_TCPIO_TIMEOUT_S;
	}
	err = getSemaphore(tcpInfo_->e.semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		throw ERR_TCPIO_TIMEOUT_S;
	}

	// 初期化処理(IPアドレス情報、ポート番号情報を取得)
	Client obj;
	try {
		obj.TcpInit(host_, service_, &tcpInfo_->hostInfo, &tcpInfo_->portInfo);
	} catch (int error) {
		throw ERR_TCPIO_SYSTEM | error;
	}

	// 送信メッセージ領域確保
	// 関数実行毎に確保する
	if (tcpInfo_->s.in != NULL) {
		// waitE()で解放が望ましいが、マルチスレッドだと確保～send間で解放されるケースがある
		delete [] tcpInfo_->s.in;
		tcpInfo_->s.in = NULL;
	}
	int msgSize = SendFixMsgSize + size; // 固定部＋可変部
	tcpInfo_->s.in = new char [msgSize];
	if (tcpInfo_->s.in == NULL) {
		throw ERR_TCPIO_MALLOC;
	}
	memset(tcpInfo_->s.in, 0, sizeof(msgSize));
	log_printf(LV_DEBUG, "msgSize = %d(%d+%d)\n", msgSize, SendFixMsgSize, size);

	// 送信メッセージデータ格納
	// フォーマットは以下
	// 8Byte：送信ヘッダ(固定文字列)
	// 4Byte：関数ID
	// 8Byte：パリティ情報
	// 4Byte：データサイズ
	// 4Byte：中間メッセージ有無情報
	// 可変 ：送信パラメータ(バイナリ形式)
	// 8Byte：送信フッタ(固定文字列)
	struct timeval time;
	getTime(&time);
	int n = 7 + size / sizeof(int); // 送信フッタインデックス
	int* ptr = reinterpret_cast<int*>(tcpInfo_->s.in);
	memcpy(ptr, "SendHead", 2 * sizeof(int));   // 送信ヘッダ("SendHead")
	*(ptr + 2) = htonl(id);                     // 関数ID
	*(ptr + 3) = htonl((int)time.tv_sec);       // パリティ情報1(現在時刻sec部分)
	*(ptr + 4) = htonl((int)time.tv_usec);      // パリティ情報2(現在時刻usec部分)
	*(ptr + 5) = htonl(size);                   // データサイズ
	*(ptr + 6) = htonl(0);                      // 中間メッセージ有無情報(なし)
	memcpy(ptr + 7, in, size);                  // 送信パラメータ
	memcpy(ptr + n, "SendFoot", 2 * sizeof(int)); // 送信フッタ("SendFoot")
	tcpInfo_->s.size  = msgSize;                // 送信メッセージサイズ

	// データ保存
	setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsWaitE, id); // E完了待ち
	tcpInfo_->parity1 = (int)time.tv_sec;  // パリティ情報1
	tcpInfo_->parity2 = (int)time.tv_usec; // パリティ情報2
	tcpInfo_->connect = connect_;          // 接続タイプ(0:常時接続型/1:毎回接続型)
	log_printf(LV_DEBUG, "parity1=%d, parity2=%d\n", tcpInfo_->parity1, tcpInfo_->parity2);

	// TCP情報をstaticに保持する。
	// クライアントが並列で実行した場合、TCP情報は処理中に上書きされてしまうため、
	// スレッド処理開始直後にauto変数に退避する。
	// 受け渡しが終るまでセマフォで保護する。
	err = getSemaphore(semId_, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		throw ERR_TCPIO_TIMEOUT_S;
	}
	staticTcpInfo_ = tcpInfo_;

	// スレッド処理開始。run()を別スレッドで実行する。
	start();

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        スレッドで実行する処理
 *               スレッド処理が並列で動作することはない
 *
 * @retval       なし
 */
void ClientTcpIo::run()
{
	const int RETRY_MAX = 30;        ///< メッセージ受信リトライ最大回数
	int       retry = 0;             ///< リトライループカウンタ
	char      header[HEADER_SIZE + 1]; ///< ヘッダ(固定文字列)
	int       funcId  = 0;           ///<  関数ID
	int       l_fid   = 0;           ///<  関数ID
	int       parity1 = 0;           ///<  パリティ情報1
	int       parity2 = 0;           ///<  パリティ情報2
	Client*    client = NULL;        ///< クライアントクラス
	bool      e_sem   = false;       ///< e.semId解放確認
	bool      th_sem  = false;       ///< thread_semId解放確認

	log_printf(LV_DEBUG, "start\n");

	// static情報をauto変数に退避。
	// TCP情報が格納されている領域は、呼び出し元でstaticに確保している前提。
	S_TcpInfo* tcpInfo = staticTcpInfo_;
	l_fid = getFuncId(tcpInfo->fid, tcpInfo->fidNum);

	// 退避中はセマフォで保護
	postSemaphore(semId_);

	try {
		client = new Client(tcpInfo->hostInfo, tcpInfo->portInfo);

		try {
			// サーバ接続
			client->TcpConnect(&tcpInfo->sd, tcpInfo->connect);
		} catch (int error) {
			throw ERR_TCPIO_CONNECT | error;
		}

		try {
			// サーバに送信メッセージ送信
			client->TcpWrite(tcpInfo->sd, tcpInfo->s.size, tcpInfo->s.in);
		} catch (int error) {
			throw ERR_TCPIO_SEND | error;
		}

		// 返信メッセージ受信
		for (retry = 0; retry < RETRY_MAX; retry++) {

			try {
				// サーバから1メッセージ読み込み、送信メッセージに対する返信メッセージかをチェック
				memset(header, 0, sizeof(header));
				client->TcpRead(tcpInfo->sd, &tcpInfo->e.size, &tcpInfo->e.out, &tcpInfo->e.rtn,
								header, &funcId, &parity1, &parity2);
			} catch (int error) {
				throw ERR_TCPIO_RECEIVE | error;
			}
			if (checkMsg(MSG_TYPE_E, l_fid, tcpInfo->parity1, tcpInfo->parity2,
						 header, funcId, parity1, parity2) == true) {
				break;
			}

			// 本スレッドがタイムアウト等で終了している場合は速やかに終了する
			if (getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle) {
				log_printf(LV_DEBUG, "this function must not continue.\n");
				throw ERR_TCPIO_FUNC_STS;
			}

			// 規定回数メッセージを読み込んでも該当しない場合はエラーとする
			if (retry == RETRY_MAX - 1) {
				log_printf(LV_ERROR, "receive retry max error.(%d)\n", retry);
				throw ERR_TCPIO_RECEIVE;
			}
		}

		// 関数の戻り値チェック
		if (tcpInfo->e.rtn != 0) {
			log_printf(LV_ERROR, "exec function rtn error.(%d)\n", tcpInfo->e.rtn);
			throw tcpInfo->e.rtn;
		}

		// 関数終了
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid); // E完了

		// サーバ切断
		client->TcpClose(&tcpInfo->sd, tcpInfo->connect);

		// セマフォ解放
		postSemaphore(tcpInfo->e.semId);
		e_sem  = true;
		th_sem = true;

		delete client;
	}
	catch (int error) {
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid);
		client->TcpClose(&tcpInfo->sd, TCP_DISCONNECT);
		tcpInfo->e.rtn = error;
		if (!e_sem)  postSemaphore(tcpInfo->e.semId);
		if (!th_sem) postSemaphore(tcpInfo->thread_semId);
		delete client;
		throw error;
	}
	catch (...) {
		log_printf(LV_ERROR, "error.\n");
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid);
		client->TcpClose(&tcpInfo->sd, TCP_DISCONNECT);
		tcpInfo->e.rtn = -1;
		if (!e_sem)  postSemaphore(tcpInfo->e.semId);
		if (!th_sem) postSemaphore(tcpInfo->thread_semId);
		delete client;
		throw - 1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        終了メッセージ受信
 * @param[in]    id          関数ID
 * @param[out]   out         出力データ
 * @param[in]    size        出力データサイズ
 * @param[in]    timeout     タイムアウト時間 [usec]
 *
 * @retval       なし
 */
void ClientTcpIo::waitE(int id, char* out, int size, int timeout)
{
	int err = 0; ///< エラー戻り値

	log_printf(LV_DEBUG, "start\n");

	// 関数実行状態チェック
	// E()完了待ちでないとE()は実行できない
	if ((getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsWaitE) &&
			(getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsFinE)) {
		log_printf(LV_DEBUG, "funcStatus is wrong.(%d)\n", getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id));
		throw ERR_TCPIO_FUNC_STS;
	}

	// セマフォ確保
	err = getSemaphore(tcpInfo_->e.semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsIdle, id);
		postSemaphore(tcpInfo_->thread_semId);
		throw ERR_TCPIO_TIMEOUT_E;
	}

	// セマフォ解放
	postSemaphore(tcpInfo_->e.semId);
	postSemaphore(tcpInfo_->thread_semId);

	// 関数実行状態変更
	setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsIdle, id);

	// 出力データ受け取り
	if (tcpInfo_->e.out != NULL) {
		memcpy(out, tcpInfo_->e.out, size);
	}

	// 関数戻り値チェック
	if (tcpInfo_->e.rtn != 0) {
		throw tcpInfo_->e.rtn;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        メッセージチェック
 * @param[in]    msgType     メッセージタイプ
 * @param[in]    inFuncId    入力関数ID
 * @param[in]    inParity1   入力パリティ情報1
 * @param[in]    inParity2   入力パリティ情報2
 * @param[in]    outHeader   出力ヘッダ
 * @param[in]    outFuncId   出力関数ID
 * @param[in]    outParity1  出力パリティ情報1
 * @param[in]    outParity2  出力パリティ情報2
 *
 * @retval       true        メッセージ一致
 * @retval       false       メッセージ不一致
 */
bool ClientTcpIo::checkMsg(E_MsgType msgType, int inFuncId, int inParity1, int inParity2,
						   char* outHeader, int outFuncId, int outParity1, int outParity2)
{
	// ヘッダが一致しているか？
	switch (msgType) {
	case MSG_TYPE_S:
		if (strcmp(outHeader, "SendHead") != 0) {
			log_printf(LV_DEBUG, "header no match(SendHead vs %s)\n", outHeader);
			return false;
		}
		break;
	case MSG_TYPE_E:
		if (strcmp(outHeader, "RecvHead") != 0) {
			log_printf(LV_DEBUG, "header no match(RecvHead vs %s)\n", outHeader);
			return false;
		}
		break;
	}

	// 関数IDが一致しているか？
	if (inFuncId != outFuncId) {
		log_printf(LV_DEBUG, "funcId no match(%d vs %d)\n", inFuncId, outFuncId);
		return false;
	}

	// パリティ情報1が一致しているか？
	if (inParity1 != outParity1) {
		log_printf(LV_DEBUG, "parity1 no match(%d vs %d)\n", inParity1, outParity1);
		return false;
	}

	// パリティ情報2が一致しているか？
	if (inParity2 != outParity2) {
		log_printf(LV_DEBUG, "parity2 no match(%d vs %d)\n", inParity2, outParity2);
		return false;
	}

	return true;
}

/**
 * @brief        関数実行状態取得
 * @param[in]    fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    funcId      関数ID
 *
 * @retval       関数実行状態(0:Idle状態/1:C完了待ち/2:E完了待ち/3:E完了)
 */
E_FuncSts ClientTcpIo::getFuncIdSts(S_TcpFuncId* fid, int fidNum, int funcId)
{
	E_FuncSts sts = E_FuncStsIdle;

	for (int i = 0; i < fidNum; i++) {
		if (funcId == fid[i].funcId) {
			sts = fid[i].sts;
		}
	}
	return sts;
}

/**
 * @brief        関数実行状態設定
 * @param[inout] fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    sts         関数実行状態(0:Idle状態/1:C完了待ち/2:E完了待ち/3:E完了)
 * @param[in]    funcId      関数ID
 *
 * @retval       なし
 */
void ClientTcpIo::setFuncIdSts(S_TcpFuncId* fid, int fidNum, E_FuncSts sts, int funcId)
{
	for (int i = 0; i < fidNum; i++) {
		if (funcId == fid[i].funcId) {
			fid[i].sts = sts;
			log_printf(LV_DEBUG, "funcStatus(fid:%d) = %d\n", i + 1, fid[i].sts);
		}
	}
	return;
}

/**
 * @brief        関数ID取得
 * @param[in]    fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 *
 * @retval       関数ID
 */
int ClientTcpIo::getFuncId(S_TcpFuncId* fid, int fidNum)
{
	int funcId = 0;

	// 実行中の関数は同時には1つしかないことを前提として
	// 現在実行中の関数IDを取得する
	for (int i = 0; i < fidNum; i++) {
		if ((fid[i].sts != E_FuncStsIdle) &&
				(fid[i].sts != E_FuncStsFinE)) {
			funcId = fid[i].funcId;
		}
	}
	return funcId;
}


----------------------------------------------------------------------------------------------------
/**
 * @file    ClientTcpIo.cpp
 * @author  Copyright(C) Canon Inc. All right reserved.
 * @date    2014/01/10
 * @brief   TCP通信処理
 * @par     説明
 *  - TCP通信のクライアント側通信部分の機能を実現するクラス
 *  - (中間メッセージなし)
 */
#include <netdb.h>
#include <cstring>
#include "IF_ClientTcpIo.hpp"
#include "Client.hpp"
#include "TcpUtil.hpp"

// クライアントが並列で実行した場合、TCP情報は処理中に上書きされてしまうため、
// staticに保存する期間は最小に抑える必要がある。
S_TcpInfo* ClientTcpIo::staticTcpInfo_;  ///< TCP情報
int        ClientTcpIo::semId_ = -1;      ///< スレッド間受け渡しパラメータの保護用セマフォID

/**
 * @brief        コンストラクタ
 * @param[in]    tcpInfo     TCP情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    host        ホスト名
 * @param[in]    service     サービス名
 * @param[in]    connect     接続タイプ(0:常時接続型/1:毎回接続型)
 */
ClientTcpIo::ClientTcpIo(S_TcpInfo* tcpInfo, int fidNum, const char* host, const char* service)
	: UtilDetachThread("TcpSendThread", run), tcpInfo_(tcpInfo), host_(host), service_(service)
{
	connect_ = TCP_CONNECT; // connectなしコンストラクタは常時接続型扱い
	if (tcpInfo_->fid == NULL) {
		tcpInfo_->fidNum = fidNum;
		tcpInfo_->fid = new S_TcpFuncId [fidNum];
		for (int i = 0; i < tcpInfo_->fidNum; i++) {
			tcpInfo_->fid[i].funcId = i + 1;
			tcpInfo_->fid[i].sts    = E_FuncStsIdle;
		}
	}
}
ClientTcpIo::ClientTcpIo(S_TcpInfo* tcpInfo, int fidNum, const char* host, const char* service, int connect)
	: UtilDetachThread("TcpSendThread", run), tcpInfo_(tcpInfo), host_(host), service_(service), connect_(connect)
{
	if (tcpInfo_->fid == NULL) {
		tcpInfo_->fidNum = fidNum;
		tcpInfo_->fid = new S_TcpFuncId [fidNum];
		for (int i = 0; i < tcpInfo_->fidNum; i++) {
			tcpInfo_->fid[i].funcId = i + 1;
			tcpInfo_->fid[i].sts    = E_FuncStsIdle;
		}
	}
}

/**
 * @brief        デストラクタ
 */
ClientTcpIo::~ClientTcpIo()
{
}

/**
 * @brief        関数実行
 * @param[in]    id          関数ID
 * @param[in]    in          入力データ
 * @param[in]    size        入力データサイズ
 * @param[in]    timeout     タイムアウト時間 [usec]
 *
 * @retval       なし
 */
void ClientTcpIo::exec(int id, char* in, int size, int timeout)
{
	int err = 0; ///< エラー戻り値

	log_printf(LV_DEBUG, "start\n");

	// 同一関数の連続実行は許可しない
	if ((getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsIdle) &&
			(getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsFinE)) {
		log_printf(LV_DEBUG, "same command can't exec.(%d)\n", id);
		throw ERR_TCPIO_FUNC_STS;
	}

	// 初回呼出はセマフォ初期化
	if (tcpInfo_->thread_semId == -1) {
		tcpInfo_->thread_semId = initSemaphore(Unlock);
	}
	if (tcpInfo_->e.semId == -1) {
		tcpInfo_->e.semId = initSemaphore(Unlock);
	}

	// スレッド間受け渡しパラメータを保護するためのセマフォ
	if (semId_ == -1) {
		semId_ = initSemaphore(Unlock);
	}

	// セマフォ確保
	err = getSemaphore(tcpInfo_->thread_semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		postSemaphore(tcpInfo_->thread_semId);
		throw ERR_TCPIO_TIMEOUT_S;
	}
	err = getSemaphore(tcpInfo_->e.semId, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		throw ERR_TCPIO_TIMEOUT_S;
	}

	// 初期化処理(IPアドレス情報、ポート番号情報を取得)
	Client obj;
	try {
		obj.TcpInit(host_, service_, &tcpInfo_->hostInfo, &tcpInfo_->portInfo);
	} catch (int error) {
		throw ERR_TCPIO_SYSTEM | error;
	}

	// 送信メッセージ領域確保
	// 関数実行毎に確保する
	if (tcpInfo_->s.in != NULL) {
		// waitE()で解放が望ましいが、マルチスレッドだと確保〜send間で解放されるケースがある
		delete [] tcpInfo_->s.in;
		tcpInfo_->s.in = NULL;
	}
	int msgSize = SendFixMsgSize + size; // 固定部＋可変部
	tcpInfo_->s.in = new char [msgSize];
	if (tcpInfo_->s.in == NULL) {
		throw ERR_TCPIO_MALLOC;
	}
	memset(tcpInfo_->s.in, 0, sizeof(msgSize));
	log_printf(LV_DEBUG, "msgSize = %d(%d+%d)\n", msgSize, SendFixMsgSize, size);

	// 送信メッセージデータ格納
	// フォーマットは以下
	// 8Byte：送信ヘッダ(固定文字列)
	// 4Byte：関数ID
	// 8Byte：パリティ情報
	// 4Byte：データサイズ
	// 4Byte：中間メッセージ有無情報
	// 可変 ：送信パラメータ(バイナリ形式)
	// 8Byte：送信フッタ(固定文字列)
	struct timeval time;
	getTime(&time);
	int n = 7 + size / sizeof(int); // 送信フッタインデックス
	int* ptr = reinterpret_cast<int*>(tcpInfo_->s.in);
	memcpy(ptr, "SendHead", 2 * sizeof(int));   // 送信ヘッダ("SendHead")
	*(ptr + 2) = htonl(id);                     // 関数ID
	*(ptr + 3) = htonl((int)time.tv_sec);       // パリティ情報1(現在時刻sec部分)
	*(ptr + 4) = htonl((int)time.tv_usec);      // パリティ情報2(現在時刻usec部分)
	*(ptr + 5) = htonl(size);                   // データサイズ
	*(ptr + 6) = htonl(0);                      // 中間メッセージ有無情報(なし)
	memcpy(ptr + 7, in, size);                  // 送信パラメータ
	memcpy(ptr + n, "SendFoot", 2 * sizeof(int)); // 送信フッタ("SendFoot")
	tcpInfo_->s.size  = msgSize;                // 送信メッセージサイズ

	// データ保存
	setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsWaitE, id); // E完了待ち
	tcpInfo_->parity1 = (int)time.tv_sec;  // パリティ情報1
	tcpInfo_->parity2 = (int)time.tv_usec; // パリティ情報2
	tcpInfo_->connect = connect_;          // 接続タイプ(0:常時接続型/1:毎回接続型)
	log_printf(LV_DEBUG, "parity1=%d, parity2=%d\n", tcpInfo_->parity1, tcpInfo_->parity2);

	// TCP情報をstaticに保持する。
	// クライアントが並列で実行した場合、TCP情報は処理中に上書きされてしまうため、
	// スレッド処理開始直後にauto変数に退避する。
	// 受け渡しが終るまでセマフォで保護する。
	err = getSemaphore(semId_, timeout);
	if (err == -1) {
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		throw ERR_TCPIO_TIMEOUT_S;
	}
	staticTcpInfo_ = tcpInfo_;

	// スレッド処理開始。run()を別スレッドで実行する。
	start();

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        スレッドで実行する処理
 *               スレッド処理が並列で動作することはない
 *
 * @retval       なし
 */
void ClientTcpIo::run()
{
	const int RETRY_MAX = 30;        ///< メッセージ受信リトライ最大回数
	int       retry = 0;             ///< リトライループカウンタ
	char      header[HEADER_SIZE + 1]; ///< ヘッダ(固定文字列)
	int       funcId  = 0;           ///<  関数ID
	int       l_fid   = 0;           ///<  関数ID
	int       parity1 = 0;           ///<  パリティ情報1
	int       parity2 = 0;           ///<  パリティ情報2
	Client*    client = NULL;        ///< クライアントクラス
	bool      e_sem   = false;       ///< e.semId解放確認
	bool      th_sem  = false;       ///< thread_semId解放確認

	log_printf(LV_DEBUG, "start\n");

	// static情報をauto変数に退避。
	// TCP情報が格納されている領域は、呼び出し元でstaticに確保している前提。
	S_TcpInfo* tcpInfo = staticTcpInfo_;
	l_fid = getFuncId(tcpInfo->fid, tcpInfo->fidNum);

	// 退避中はセマフォで保護
	postSemaphore(semId_);

	try {
		client = new Client(tcpInfo->hostInfo, tcpInfo->portInfo);

		try {
			// サーバ接続
			client->TcpConnect(&tcpInfo->sd, tcpInfo->connect);
		} catch (int error) {
			throw ERR_TCPIO_CONNECT | error;
		}

		// 目的: 接続待ち中に呼び出し元がタイムアウトした場合、接続成功後の不要な送受信を防ぐ。
		// 処理: 都度接続モードに限り、関数状態がIdleならタイムアウト済みとして例外経路へ移す。
		if ((tcpInfo->connect == TCP_DISCONNECT) &&
				// 呼び出し側が待機を打ち切った状態か確認する。
				(getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle)) {
			// 共通の例外処理にFDのクローズとセマフォ後始末を任せる。
			throw ERR_TCPIO_FUNC_STS;
		}

		try {
			// サーバに送信メッセージ送信
			client->TcpWrite(tcpInfo->sd, tcpInfo->s.size, tcpInfo->s.in);
		} catch (int error) {
			throw ERR_TCPIO_SEND | error;
		}

		// 目的: 送信中に呼び出し元がタイムアウトした場合、応答受信待ちへ進まないようにする。
		// 処理: 都度接続モードでIdleを検出したら、例外経路で使用中のソケットを閉じる。
		if ((tcpInfo->connect == TCP_DISCONNECT) &&
				// 接続中に呼び出し側がタイムアウト状態へ変更したか確認する。
				(getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle)) {
			// 送受信を打ち切り、共通のクローズ処理へ移る。
			throw ERR_TCPIO_FUNC_STS;
		}

		// 返信メッセージ受信
		for (retry = 0; retry < RETRY_MAX; retry++) {

			try {
				// サーバから1メッセージ読み込み、送信メッセージに対する返信メッセージかをチェック
				memset(header, 0, sizeof(header));
				client->TcpRead(tcpInfo->sd, &tcpInfo->e.size, &tcpInfo->e.out, &tcpInfo->e.rtn,
								header, &funcId, &parity1, &parity2);
			} catch (int error) {
				throw ERR_TCPIO_RECEIVE | error;
			}
			if (checkMsg(MSG_TYPE_E, l_fid, tcpInfo->parity1, tcpInfo->parity2,
						 header, funcId, parity1, parity2) == true) {
				break;
			}

			// 本スレッドがタイムアウト等で終了している場合は速やかに終了する
			if (getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle) {
				log_printf(LV_DEBUG, "this function must not continue.\n");
				throw ERR_TCPIO_FUNC_STS;
			}

			// 規定回数メッセージを読み込んでも該当しない場合はエラーとする
			if (retry == RETRY_MAX - 1) {
				log_printf(LV_ERROR, "receive retry max error.(%d)\n", retry);
				throw ERR_TCPIO_RECEIVE;
			}
		}

		// 関数の戻り値チェック
		if (tcpInfo->e.rtn != 0) {
			log_printf(LV_ERROR, "exec function rtn error.(%d)\n", tcpInfo->e.rtn);
			throw tcpInfo->e.rtn;
		}

		// 関数終了
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid); // E完了

		// サーバ切断
		client->TcpClose(&tcpInfo->sd, tcpInfo->connect);

		// セマフォ解放
		postSemaphore(tcpInfo->e.semId);
		e_sem  = true;
		th_sem = true;

		delete client;
	}
	catch (int error) {
		// 整数エラーの発生を記録し、接続・送受信で確保した資源を解放する。
		log_printf(LV_ERROR, "error.(0x%x)\n", error);
		// 目的: タイムアウト側とワーカースレッド側の二重解放やFDリークを防止する。
		// 現在IdleならwaitE側がすでにタイムアウト処理を始めていることを記録する。
		bool callerTimedOut = (getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle);
		// 呼び出し元へエラー完了を知らせるため、関数状態を完了状態へ更新する。
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid);
		// clientとFDの両方が有効な場合に限り、都度接続したソケットを閉じる。
		if ((client != NULL) && (tcpInfo->sd >= 0)) {
			// TCP_DISCONNECTを指定してshutdown/closeし、共有FDも無効化する。
			client->TcpClose(&tcpInfo->sd, TCP_DISCONNECT);
		}
		// waitEへ今回の整数エラーを返すため、結果領域に保存する。
		tcpInfo->e.rtn = error;
		// E完了セマフォをまだ通知していない場合にだけ、待ち側へ通知する。
		if (!e_sem)  postSemaphore(tcpInfo->e.semId);
		// タイムアウト側がすでにthreadセマフォを解放していない場合に限り、ここで解放する。
		if (!th_sem && !callerTimedOut) postSemaphore(tcpInfo->thread_semId);
		// ワーカースレッドが作成したClientオブジェクトを破棄する。
		delete client;
		// 元の整数エラーを維持して上位へ通知する。
		throw error;
	}
	catch (...) {
		// 整数以外の例外も記録し、同じ資源解放経路で安全に後始末する。
		log_printf(LV_ERROR, "error.\n");
		// 目的: 想定外の例外でもFDリークとセマフォ二重解放を防止する。
		// 現在IdleならwaitE側がタイムアウト処理中なので、その状態を保持する。
		bool callerTimedOut = (getFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, l_fid) == E_FuncStsIdle);
		// 呼び出し元に処理終了を知らせるため、関数状態を完了状態へ更新する。
		setFuncIdSts(tcpInfo->fid, tcpInfo->fidNum, E_FuncStsFinE, l_fid);
		// clientとFDが有効な場合だけ、都度接続したソケットを閉じる。
		if ((client != NULL) && (tcpInfo->sd >= 0)) {
			// TCP_DISCONNECTを指定してソケットを閉じ、共有FDを無効化する。
			client->TcpClose(&tcpInfo->sd, TCP_DISCONNECT);
		}
		// 汎用エラーを呼び出し元へ返すため、結果領域に保存する。
		tcpInfo->e.rtn = -1;
		// E完了セマフォが未通知の場合だけ、待ち側へ通知する。
		if (!e_sem)  postSemaphore(tcpInfo->e.semId);
		// タイムアウト側がthreadセマフォを解放済みなら、二重解放しない。
		if (!th_sem && !callerTimedOut) postSemaphore(tcpInfo->thread_semId);
		// ワーカースレッドが作成したClientオブジェクトを破棄する。
		delete client;
		// 想定外の例外を既存の汎用エラー値へ変換して上位へ返す。
		throw - 1;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        終了メッセージ受信
 * @param[in]    id          関数ID
 * @param[out]   out         出力データ
 * @param[in]    size        出力データサイズ
 * @param[in]    timeout     タイムアウト時間 [usec]
 *
 * @retval       なし
 */
void ClientTcpIo::waitE(int id, char* out, int size, int timeout)
{
	int err = 0; ///< エラー戻り値

	log_printf(LV_DEBUG, "start\n");

	// 関数実行状態チェック
	// E()完了待ちでないとE()は実行できない
	if ((getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsWaitE) &&
			(getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id) != E_FuncStsFinE)) {
		log_printf(LV_DEBUG, "funcStatus is wrong.(%d)\n", getFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, id));
		throw ERR_TCPIO_FUNC_STS;
	}

	// セマフォ確保
	err = getSemaphore(tcpInfo_->e.semId, timeout);
	if (err == -1) {
		// 指定時間内にE完了セマフォを取得できなかったことを記録する。
		log_printf(LV_ERROR, "timeout error.(%d)\n", timeout);
		// ワーカースレッドにタイムアウトを知らせ、受信ループを終了させる。
		setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsIdle, id);
		// 目的: E待ちタイムアウト後にブロッキングrecvを解除し、ワーカーによるFDクローズを促す。
		// 常時接続ソケットは閉じず、都度接続モードで有効なFDがある場合だけ対象とする。
		if ((connect_ == TCP_DISCONNECT) && (tcpInfo_->sd >= 0)) {
			// ソケットの送受信を停止して、ワーカースレッドのブロッキングI/Oを解除する。
			err = shutdown(tcpInfo_->sd, SHUT_RDWR);
			// shutdown失敗を記録するが、タイムアウト通知と後続処理は続行する。
			if (err < 0) {
				// FD番号とOSエラー番号をログに残し、解除失敗を見えるようにする。
				log_printf(LV_ERROR, "shutdown error.(err = %d, errno = %d)\n", err, errno);
			}
		}
		// 呼び出し側が保持しているthreadセマフォを解放する。
		postSemaphore(tcpInfo_->thread_semId);
		// E待ちタイムアウトを呼び出し側へ通知する。
		throw ERR_TCPIO_TIMEOUT_E;
	}

	// セマフォ解放
	postSemaphore(tcpInfo_->e.semId);
	postSemaphore(tcpInfo_->thread_semId);

	// 関数実行状態変更
	setFuncIdSts(tcpInfo_->fid, tcpInfo_->fidNum, E_FuncStsIdle, id);

	// 出力データ受け取り
	if (tcpInfo_->e.out != NULL) {
		memcpy(out, tcpInfo_->e.out, size);
	}

	// 関数戻り値チェック
	if (tcpInfo_->e.rtn != 0) {
		throw tcpInfo_->e.rtn;
	}

	log_printf(LV_DEBUG, "end\n");

	return;
}

/**
 * @brief        メッセージチェック
 * @param[in]    msgType     メッセージタイプ
 * @param[in]    inFuncId    入力関数ID
 * @param[in]    inParity1   入力パリティ情報1
 * @param[in]    inParity2   入力パリティ情報2
 * @param[in]    outHeader   出力ヘッダ
 * @param[in]    outFuncId   出力関数ID
 * @param[in]    outParity1  出力パリティ情報1
 * @param[in]    outParity2  出力パリティ情報2
 *
 * @retval       true        メッセージ一致
 * @retval       false       メッセージ不一致
 */
bool ClientTcpIo::checkMsg(E_MsgType msgType, int inFuncId, int inParity1, int inParity2,
						   char* outHeader, int outFuncId, int outParity1, int outParity2)
{
	// ヘッダが一致しているか？
	switch (msgType) {
	case MSG_TYPE_S:
		if (strcmp(outHeader, "SendHead") != 0) {
			log_printf(LV_DEBUG, "header no match(SendHead vs %s)\n", outHeader);
			return false;
		}
		break;
	case MSG_TYPE_E:
		if (strcmp(outHeader, "RecvHead") != 0) {
			log_printf(LV_DEBUG, "header no match(RecvHead vs %s)\n", outHeader);
			return false;
		}
		break;
	}

	// 関数IDが一致しているか？
	if (inFuncId != outFuncId) {
		log_printf(LV_DEBUG, "funcId no match(%d vs %d)\n", inFuncId, outFuncId);
		return false;
	}

	// パリティ情報1が一致しているか？
	if (inParity1 != outParity1) {
		log_printf(LV_DEBUG, "parity1 no match(%d vs %d)\n", inParity1, outParity1);
		return false;
	}

	// パリティ情報2が一致しているか？
	if (inParity2 != outParity2) {
		log_printf(LV_DEBUG, "parity2 no match(%d vs %d)\n", inParity2, outParity2);
		return false;
	}

	return true;
}

/**
 * @brief        関数実行状態取得
 * @param[in]    fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    funcId      関数ID
 *
 * @retval       関数実行状態(0:Idle状態/1:C完了待ち/2:E完了待ち/3:E完了)
 */
E_FuncSts ClientTcpIo::getFuncIdSts(S_TcpFuncId* fid, int fidNum, int funcId)
{
	E_FuncSts sts = E_FuncStsIdle;

	for (int i = 0; i < fidNum; i++) {
		if (funcId == fid[i].funcId) {
			sts = fid[i].sts;
		}
	}
	return sts;
}

/**
 * @brief        関数実行状態設定
 * @param[inout] fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 * @param[in]    sts         関数実行状態(0:Idle状態/1:C完了待ち/2:E完了待ち/3:E完了)
 * @param[in]    funcId      関数ID
 *
 * @retval       なし
 */
void ClientTcpIo::setFuncIdSts(S_TcpFuncId* fid, int fidNum, E_FuncSts sts, int funcId)
{
	for (int i = 0; i < fidNum; i++) {
		if (funcId == fid[i].funcId) {
			fid[i].sts = sts;
			log_printf(LV_DEBUG, "funcStatus(fid:%d) = %d\n", i + 1, fid[i].sts);
		}
	}
	return;
}

/**
 * @brief        関数ID取得
 * @param[in]    fid         関数ID情報
 * @param[in]    fidNum      関数ID数
 *
 * @retval       関数ID
 */
int ClientTcpIo::getFuncId(S_TcpFuncId* fid, int fidNum)
{
	int funcId = 0;

	// 実行中の関数は同時には1つしかないことを前提として
	// 現在実行中の関数IDを取得する
	for (int i = 0; i < fidNum; i++) {
		if ((fid[i].sts != E_FuncStsIdle) &&
				(fid[i].sts != E_FuncStsFinE)) {
			funcId = fid[i].funcId;
		}
	}
	return funcId;
}


----------------------------------------------------------------------------------------------------
