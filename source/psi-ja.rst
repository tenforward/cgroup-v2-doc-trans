.. _psi:

================================
PSI - Pressure Stall Information
================================

:Date: April, 2018
:Author: Johannes Weiner <hannes@cmpxchg.org>

..
  When CPU, memory or IO devices are contended, workloads experience
  latency spikes, throughput losses, and run the risk of OOM kills.
..

CPU、メモリー、IO デバイスが競合すると、ワークロードはレイテンシーの急上昇、スループットの低下、OOM による強制終了のリスクに直面します。

..
  Without an accurate measure of such contention, users are forced to
  either play it safe and under-utilize their hardware resources, or
  roll the dice and frequently suffer the disruptions resulting from
  excessive overcommit.
..

このような競合を正確に測定しないと、ユーザーは安全策をとってハードウェア リソースを十分に活用しないか、運を天に任せて過剰なオーバーコミットによる中断を頻繁に経験するかのいずれかを余儀なくされます。

..
  The psi feature identifies and quantifies the disruptions caused by
  such resource crunches and the time impact it has on complex workloads
  or even entire systems.
..

PSI 機能は、このようなリソース不足によって引き起こされる混乱と、複雑なワークロードやシステム全体に及ぼす時間的影響を特定して定量化します。

..
  Having an accurate measure of productivity losses caused by resource
  scarcity aids users in sizing workloads to hardware--or provisioning
  hardware according to workload demand.
..

リソース不足によって生じる生産性の損失を正確に測定することで、ユーザーはハードウェアに合わせてワークロードのサイズを決定したり、ワークロードの需要に応じてハードウェアをプロビジョニングしたりできるようになります。

..
  As psi aggregates this information in realtime, systems can be managed
  dynamically using techniques such as load shedding, migrating jobs to
  other systems or data centers, or strategically pausing or killing low
  priority or restartable batch jobs.
..

psi はこの情報をリアルタイムで集約するため、負荷制限、他のシステムやデータ センターへのジョブの移行、優先度の低いバッチ ジョブや再開可能なバッチ ジョブの戦略的な一時停止や強制終了などの手法を使用して、システムを動的に管理できます。

..
  This allows maximizing hardware utilization without sacrificing
  workload health or risking major disruptions such as OOM kills.
..

これにより、ワークロードの健全性を犠牲にしたり、OOM kill などの大きな中断のリスクを冒したりすることなく、ハードウェアの使用率を最大化できます。

..
  Pressure interface
  ==================
圧力インターフェース
================

..
  Pressure information for each resource is exported through the
  respective file in /proc/pressure/ -- cpu, memory, and io.
..

各リソースの圧力情報 (CPU、メモリ、IO) は、/proc/pressure/ 内のそれぞれのファイルを通じてエクスポートされます。

..
  The format is as such::
..

形式は次の通りです::

	some avg10=0.00 avg60=0.00 avg300=0.00 total=0
	full avg10=0.00 avg60=0.00 avg300=0.00 total=0

..
  The "some" line indicates the share of time in which at least some
  tasks are stalled on a given resource.
..

"some" 行は、特定のリソース上で少なくとも一部のタスクがストールしている時間の割合を示します。

..
  The "full" line indicates the share of time in which all non-idle
  tasks are stalled on a given resource simultaneously. In this state
  actual CPU cycles are going to waste, and a workload that spends
  extended time in this state is considered to be thrashing. This has
  severe impact on performance, and it's useful to distinguish this
  situation from a state where some tasks are stalled but the CPU is
  still doing productive work. As such, time spent in this subset of the
  stall state is tracked separately and exported in the "full" averages.
..

"full" 行は、アイドル状態ではないすべてのタスクが特定のリソースで同時にストールしている時間の割合を示します。この状態では、実際の CPU サイクルが無駄になり、この状態で長時間を過ごすワークロードはスラッシングと見なされます。これはパフォーマンスに重大な影響を与えるため、一部のタスクがストールしているが CPU がまだ生産的な作業を行っている状態とこの状況を区別することは有用です。そのため、ストール状態のこのサブセットで費やされた時間は個別に追跡され、"full" 平均でエクスポートされます。

..
  CPU full is undefined at the system level, but has been reported
  since 5.13, so it is set to zero for backward compatibility.
..

CPU フルは、システムレベルでは未定義ですが、5.13 以降報告されるので、下位互換性のためにゼロに設定されています。

..
  The ratios (in %) are tracked as recent trends over ten, sixty, and
  three hundred second windows, which gives insight into short term events
  as well as medium and long term trends. The total absolute stall time
  (in us) is tracked and exported as well, to allow detection of latency
  spikes which wouldn't necessarily make a dent in the time averages,
  or to average trends over custom time frames.
..

比率 (%) は、10 秒、60 秒、300 秒のウィンドウの最近の傾向として追跡され、短期的なイベントだけでなく、中期および長期の傾向についての洞察が得られます。合計絶対ストール時間 (us) も追跡およびエクスポートされ、必ずしも時間平均に影響を与えないレイテンシスパイクを検出したり、カスタム時間枠の傾向を平均化したりすることができます。

..
  Monitoring for pressure thresholds
  ==================================
..
圧力閾値のモニタリング
==================

..
  Users can register triggers and use poll() to be woken up when resource
  pressure exceeds certain thresholds.
..

ユーザーはトリガーを登録し、poll() を使用して、リソースの負荷が特定のしきい値を超えたときに起動されるようにすることができます。

..
  A trigger describes the maximum cumulative stall time over a specific
  time window, e.g. 100ms of total stall time within any 500ms window to
  generate a wakeup event.
..

トリガーは、特定のタイムウィンドウ内の最大累積ストール時間を表します。たとえば、ウェイクアップ イベントを生成するための 500 ミリ秒のウィンドウ内の合計ストール時間は 100 ミリ秒です。

..
  To register a trigger user has to open psi interface file under
  /proc/pressure/ representing the resource to be monitored and write the
  desired threshold and time window. The open file descriptor should be
  used to wait for trigger events using select(), poll() or epoll().
  The following format is used::
..

トリガーを登録するには、ユーザーは監視対象のリソースを表す /proc/pressure/ の psi インターフェイス ファイルを開き、必要なしきい値とタイムウィンドウを書き込む必要があります。開いているファイル記述子は、select()、poll()、または epoll() を使用してトリガー イベントを待機するために使用する必要があります。次の形式が使用されます。

	<some|full> <stall amount in us> <time window in us>

..
  For example writing "some 150000 1000000" into /proc/pressure/memory
  would add 150ms threshold for partial memory stall measured within
  1sec time window. Writing "full 50000 1000000" into /proc/pressure/io
  would add 50ms threshold for full io stall measured within 1sec time window.
..

たとえば、/proc/pressure/memory に "some 150000 1000000"と書き込むと、1 秒のタイムウィンドウ内で計測された部分的なメモリーのストールの 150ms というしきい値が追加されます。/proc/pressure/io に "full 50000 1000000" と書き込むと、1 秒のタイムウィンドウ内で測定された完全（full）な IO ストールの 50ms というしきい値が追加されます。

..
  Triggers can be set on more than one psi metric and more than one trigger
  for the same psi metric can be specified. However for each trigger a separate
  file descriptor is required to be able to poll it separately from others,
  therefore for each trigger a separate open() syscall should be made even
  when opening the same psi interface file. Write operations to a file descriptor
  with an already existing psi trigger will fail with EBUSY.
..

トリガーは複数の psi メトリックに設定でき、同じ psi メトリックに対して複数のトリガーを指定できます。ただし、トリガーごとに、他のトリガーとは別にポーリングできるようにするために個別のファイル記述子が必要です。したがって、同じ psi インターフェイス ファイルを開く場合でも、トリガーごとに個別の open() システムコールを実行する必要があります。既存の psi トリガーを持つファイル記述子への書き込み操作は、EBUSY で失敗します。

..
  Monitors activate only when system enters stall state for the monitored
  psi metric and deactivates upon exit from the stall state. While system is
  in the stall state psi signal growth is monitored at a rate of 10 times per
  tracking window.
..

モニターは、システムが監視対象の PSI メトリックのストール状態に入ったときにのみアクティブになり、ストール状態から出ると非アクティブになります。システムがストール状態にある間、PSI 信号の増加は追跡ウィンドウごとに 10 回の割合で監視されます。

..
  The kernel accepts window sizes ranging from 500ms to 10s, therefore min
  monitoring update interval is 50ms and max is 1s. Min limit is set to
  prevent overly frequent polling. Max limit is chosen as a high enough number
  after which monitors are most likely not needed and psi averages can be used
  instead.
..

カーネルは 500 ミリ秒から 10 秒の範囲のウィンドウサイズを受け入れるため、最小の監視更新間隔は 50 ミリ秒、最大は 1 秒です。最小制限は、ポーリングが過度に頻繁に行われないようにするために設定されます。最大制限は、十分に高い数値として選択され、それを超えると、モニターはほとんど必要なくなり、代わりに PSI 平均を使用できます。

..
  Unprivileged users can also create monitors, with the only limitation that the
  window size must be a multiple of 2s, in order to prevent excessive resource
  usage.
..

非特権ユーザーもモニターを作成できますが、リソースの過剰な使用を防ぐために、ウィンドウサイズは 2 秒の倍数でなければならないという唯一の制限があります。

..
  When activated, psi monitor stays active for at least the duration of one
  tracking window to avoid repeated activations/deactivations when system is
  bouncing in and out of the stall state.
..

アクティブ化されると、psi モニターは少なくとも 1 つの追跡ウィンドウの期間中はアクティブのままになり、システムがストール状態になったり抜けたりするときにアクティブ化/非アクティブ化が繰り返されるのを防ぎます。

..
  Notifications to the userspace are rate-limited to one per tracking window.
..

ユーザー空間への通知は、追跡ウィンドウごとに 1 回に制限されます。

..
  The trigger will de-register when the file descriptor used to define the
  trigger  is closed.
..

トリガーを定義するために使用されたファイル記述子が閉じられると、トリガーは登録解除されます。

..
  Userspace monitor usage example
  ===============================
ユーザー空間モニターの使用例
=======================

::

  #include <errno.h>
  #include <fcntl.h>
  #include <stdio.h>
  #include <poll.h>
  #include <string.h>
  #include <unistd.h>

  /*
   * Monitor memory partial stall with 1s tracking window size
   * and 150ms threshold.
   */
  int main() {
	const char trig[] = "some 150000 1000000";
	struct pollfd fds;
	int n;

	fds.fd = open("/proc/pressure/memory", O_RDWR | O_NONBLOCK);
	if (fds.fd < 0) {
		printf("/proc/pressure/memory open error: %s\n",
			strerror(errno));
		return 1;
	}
	fds.events = POLLPRI;

	if (write(fds.fd, trig, strlen(trig) + 1) < 0) {
		printf("/proc/pressure/memory write error: %s\n",
			strerror(errno));
		return 1;
	}

	printf("waiting for events...\n");
	while (1) {
		n = poll(&fds, 1, -1);
		if (n < 0) {
			printf("poll error: %s\n", strerror(errno));
			return 1;
		}
		if (fds.revents & POLLERR) {
			printf("got POLLERR, event source is gone\n");
			return 0;
		}
		if (fds.revents & POLLPRI) {
			printf("event triggered!\n");
		} else {
			printf("unknown event received: 0x%x\n", fds.revents);
			return 1;
		}
	}

	return 0;
  }

..
  Cgroup2 interface
  =================
cgroup2 インターフェース
=====================

..
  In a system with a CONFIG_CGROUPS=y kernel and the cgroup2 filesystem
  mounted, pressure stall information is also tracked for tasks grouped
  into cgroups. Each subdirectory in the cgroupfs mountpoint contains
  cpu.pressure, memory.pressure, and io.pressure files; the format is
  the same as the /proc/pressure/ files.
..

CONFIG_CGROUPS=y と設定されたカーネルと cgroup2 ファイルシステムがマウントされているシステムでは、cgroup にグループ化されたタスクの圧力ストール情報も追跡されます。cgroupfs マウントポイントの各サブディレクトリには、cpu.pressure、memory.pressure、および io.pressure ファイルが含まれており、その形式は /proc/pressure/ ファイルと同じです。

..
  Per-cgroup psi monitors can be specified and used the same way as
  system-wide ones.
..

cgroup ごとの psi モニターは、システム全体のモニターと同じように指定して使用できます。
