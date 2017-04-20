注)  本文書は英語から翻訳したものであり、その内容が最新でない場合もあります。最新の情報はオリジナルの英語版を参照してください。

変更履歴とその他の情報は http://aka.ms/AdaptiveIndexDefrag にあります。

**AdaptiveIndexDefrag の目的は何?**
このプロシージャの目的は、複数のデータベースの複数のインデックスをインテリジェントにデフラグすることです。 簡単に言えば、このプロシージャは、索引を再構築するか再構成するかを自動的に選択します。 判断基準は断片化の度合いと、ページロックが許可されているか、LOBが存在するかなどのその他のパラメータです。 制限時間 (デフォルトは 8 時間) で処理を打ち切ります。 デフラグの優先度を、サイズ、断片化の度合い、または (範囲スキャンカウントに基づく) インデックス使用率 (これがデフォルトです) のいずれかで設定することもできます。 また、パーティション化されたインデックス、統計の更新 (テーブル全体またはインデックスに関連するもののみを選択可能)、独自のフィルファクターを指定しての再構築、およびオンライン操作での再構築などのオプションを指定できます。

**使用可能な SQL Server のバージョンは?**
このプロシージャは、DMV と DMF が関係するため、SQL Server 2005 SP2 以降で使用できます。

**どうやってデプロイするの?**
v1.3.7 以降では、usp_AdaptiveIndexDefrag (とそのサポートオブジェクト) を作成したい任意のデータベースコンテキストを選択して、添付されたスクリプトを開きます。一番上にある @deploymode 変数をそのままにしておけば、アップグレードモード (すべての履歴データを保持する) になります。 変数の値を書き換えれば、古いバージョンとオブジェクトを上書き (履歴データを無視する) して新規にデプロイすることができます。

**どうやって使うの?**
任意のユーザデータベース上で、添付のスクリプトを実行した後、パラメータなしでプロシージャ usp_AdaptiveIndexDefrag を実行するか、すべて省略 (指定しない場合は各パラメータのデフォルトが使用されます) するか、パラメータをカスタマイズします。 OPTIONS.md ファイルを参照して、使用可能なすべてのパラメータを確認して下さい。

**添付のスクリプトを実行すると何が作成されるの?**

- いくつかの制御テーブルとログ・テーブルが作成されます:
  - tbl_AdaptiveIndexDefrag_Working は、どのオブジェクトを操作するかを把握するために使われます。 オブジェクトの処理方法に影響する重要な情報です。 また、デフラグ・サイクルが時間の制約のために数日に及ぶ場合、前回の実行でどのインデックスがデフラグ済みになったかを記録します。
  - tbl_AdaptiveIndexDefrag_Stats_Working の用途は上記のテーブルと同様ですが、インデックスではなく統計を扱います。
  - tbl_AdaptiveIndexDefrag_log は、インデックスのログ・テーブルです。全てのインデックスへの操作がログに記録されます。
  - tbl_AdaptiveIndexDefrag_Stats_log は、統計のログ・テーブルです。全ての統計への操作がログに記録されます。 usp_AdaptiveIndexDefrag_PurgeLogs プロシージャを使用することで、 tbl_AdaptiveIndexDefrag_log と tbl_AdaptiveIndexDefrag_Stats_log をクリーンアップすることができます。
  - tbl_AdaptiveIndexDefrag_Exceptions は、特定のオブジェクトが処理される日を制限できる例外テーブルです (sysschedules システムテーブルと同様のマスクを指定します)。 また、特定のインデックス、テーブルまたはデータベース全体に対して例外を設定することもできます。 週末にのみデフラグする特定のテーブルがあるとします。そのテーブル上の全てのインデックスが土曜日と日曜日にデフラグされるように例外テーブルに設定できます。 または、1 つのデータベースまたはテーブルを最適化から除外します。 これらは、特定のニーズに対処する方法の一例です。
  - tbl_AdaptiveIndexDefrag_IxDisableStatus は、無効化されたインデックスが記録される場所です。 これにより、デフラグ・サイクルを中断したとき、インデックスが人の手ではなくデフラグ・サイクルそのものによって無効化されたことを把握できます。
  - usp_AdaptiveIndexDefrag_PurgeLogs は、90 日以上経過したログ・テーブルをパージします。 これにより無制限の増加を防ぐことができます。 90 日は単なるデフォルト値です。 @daystokeep 入力パラメータを適切だと考える値に変更して下さい。 ジョブの中でこれを実行することをお勧めします。
  - usp_AdaptiveIndexDefrag_Exclusions は、特定のインデックスまたは特定のテーブルの全てのインデックスのデフラグを許可する日 (もしあれば) を設定するのに役立ちます。 以前は、スクリプトに埋め込まれた除外項目を設定する方法の例を記載していましたが、フィードバックによってストアド・プロシージャに変更しました。
  - usp_AdaptiveIndexDefrag_CurrentExecStats を使うと、今までにデフラグされたインデックスを把握できます。
  - usp_AdaptiveIndexDefrag は、インデックスの断片化と統計情報の更新を処理するメインプロシージャです。 前に示した入力パラメータを受け取ります。
- その他の目的のためのいくつかのビューが作成されます:
  - vw_ErrLst30Days で、過去 30 日間の全ての既知の実行エラーを確認できます。
  - vw_ErrLst24Hrs で、過去 24 時間の全ての既知の実行エラーを確認できます。
  - vw_AvgTimeLst30Days で、過去 30 日間の、各インデックスを処理するのにかかった平均実行時間を確認できます。
  - vw_AvgFragLst30Days で、過去 30 日間の、各インデックスで検出された平均の断片化状況を確認できます。
  - vw_AvgLargestLst30Days で、過去 30 日間の、各インデックスの平均サイズを確認できます。
  - vw_AvgMostUsedLst30Days で、過去 30 日間の、各インデックスの平均使用率を確認できます。
  - vw_LastRun_Log で、最後の実行で処理された内容を確認できます。

このスクリプトの一般的な使用例をいくつか示します:

**EXEC dbo.usp_AdaptiveIndexDefrag**
デフォルトは、

- 断片化が5% を超えるインデックスをデフラグします。
- 断片化が30% を超えるインデックスを再構築します。
- すべてのインデックスが対象です。
- コマンドは自動的に実行されます。
- RANGE_SCAN_COUNT 値の降順でインデックスをデフラグします。
- 制限時間は 480 分 (8 時間) です。
- すべてのデータベースが対象です。
- すべてのテーブルが対象です。
- インデックスを再スキャンします。
- スキャンは LIMITED モードで実行されます。
- LOB は圧縮されます。
- デフラグは 8 ページ以上あるインデックスに限定されます。
- インデックスはオフラインでデフラグされます。
- インデックスは (訳注: TempDBではなく、そのインデックスが存在する) データベース上でソートされます。
- インデックスのフィルファクターは元の値が維持されます。
- 8 ページ以上の場合は、最も右側に配置されたパーティションのみが考慮されます。
- 統計は再構成されたインデックスで更新されます。
- デフラグはプロセッサのシステムデフォルトを使用します。
- t-sql コマンド文字列を出力しません。
- 断片化の度合いを出力しません。
- インデックスへの操作の間に 5 秒待ちます。

**EXEC dbo.usp_AdaptiveIndexDefrag @dbScope = 'AdventureWorks2014'**
スコープが 'AdventureWorks2014' データベースのみである点を除き、上記と同じです。

**EXEC dbo.usp_AdaptiveIndexDefrag @dbScope = 'AdventureWorks2014', @tblName = 'Production.BillOfMaterials'**
上記と同じですが、BillOfMaterials テーブルのみを対象とします。

**EXEC dbo.usp_AdaptiveIndexDefrag @Exec_Print = 0, @printCmds = 1**
@Exec_Print を0にするとコマンドは実行されません。 代わりに、コマンドを画面に表示するだけになります。 舞台裏で何をしているのかを確認したい場合に便利です。

**EXEC dbo.usp_AdaptiveIndexDefrag @Exec_Print = 0, @printCmds = 1, @scanMode = 'DETAILED', @updateStatsWhere = 1**
上記と同じですが、 @scanMode の DETAILED 指定を追加して、統計を更新するためのしきい値を細かくします。 さらに、テーブル上の全ての統計ではなく、インデックスに関連する統計のみを更新するようにします。

**EXEC dbo.usp_AdaptiveIndexDefrag @scanMode = 'DETAILED', @updateStatsWhere = 1 , @disableNCIX = 1**
コマンドを表示するのではなく実行し、再構築の前に非クラスタ化インデックスを無効にするのが上記と異なります。

**EXEC dbo.usp_AdaptiveIndexDefrag @minFragmentation = 3, @rebuildThreshold = 20**
これは、デフラグ対象にするインデックスの断片化率の最小値を 3% に、再構築と再構成のしきい値を 20% に下げます。

**EXEC dbo.usp_AdaptiveIndexDefrag @onlineRebuild = 1**
これは可能な限りオンラインで再構築を実行しようとします。

**EXEC dbo.usp_AdaptiveIndexDefrag @onlineRebuild = 1, @updateStatsWhere = 0, @statsSample = 'FULLSCAN'**
上記と似ていますが、FULLSCAN を指定してテーブル上の全統計情報を更新します。

**EXEC dbo.usp_AdaptiveIndexDefrag @onlineRebuild = 1, @updateStatsWhere = 0, @dbScope = 'AdventureWorks2014', @defragOrderColumn = 'fragmentation', @timeLimit = 240, @scanMode = 'DETAILED'**
上記と似ていますが、これはすべてのデフラグ操作を 'AdventureWorks2014' データベースに制限します。 最も断片化されたインデックスを優先します (デフォルトの使用率の高い順ではなく)。 デフラグ処理のタイムリミットを 4 時間に制限します。 @scanMode の DETAILED 指定を追加して、統計を更新するためのしきい値を細かくします。

**EXEC dbo.usp_AdaptiveIndexDefrag @timeLimit = 360**
制限時間を 360 分 (6 時間) に設定します。

**EXEC dbo.usp_AdaptiveIndexDefrag @offlinelocktimeout = 300, @onlinelocktimeout = 5**
オフラインリビルドを実行する場合は 300 秒、オンラインリビルドを実行する場合は 5 分 (SQL Server 2014以降で有効) のロックタイムアウト値を設定します。

**EXEC dbo.usp_AdaptiveIndexDefrag @rebuildThreshold = 99, @dealMaxPartition = 1, @onlineRebuild = 1**

- 可能であれば、オンラインでの再構築を実行しようとします。
- 最も右のパーティションをデフラグ実行から除外します。 再構築のしきい値を 99% に設定することで、実質的に再構築の代わりに再編成を強制します。
- 何らかの目的により、パーティション化されたテーブルで、全てのインデックスを低いフィルファクターで作成しているとき、より高いフィルファクターを使用してインデックスを再構築し、DBCC SHRINKFILE を使用してスペースを回収しなければならない場合に役立ちます (はい、 これはお勧めできないのですが、時々起こりうることです)。
- 右端のパーティション (アクティブ) を除くすべてのパーティションで強制的に再編成することは、サーバーの可用性への影響を最小限に抑えながら、インデックスを最適化する方法です。
