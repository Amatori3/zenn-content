---
title: "Flutter Web の CanvasKit と skwasm、実測して比較してみた"
emoji: "🎨"
type: "tech"
topics: ["flutter", "flutterweb", "canvaskit", "webassembly", "docker"]
published: true
---

## TL;DR

- 初回ロードは skwasm の方が軽くて速い(転送量 4.0MB vs 4.8MB、全リクエスト
  完了 795ms vs 1.19s)
- ランタイム性能は一方的ではない: **build(ウィジェットツリー構築)は
  skwasmが、raster(実際の描画)はCanvasKitが、3シナリオ全てで一貫して
  優位**。総合fpsはどちらの処理が支配的かで勝敗が変わる
- 今回のベンチアプリでは、raster寄りの処理(DataTableスクロール・画面
  遷移)はCanvasKitが、build寄りの処理(フォーム入力)はskwasmが
  総合fpsで上回った
- おまけ: `--wasm` ビルドは Firefox だと黙って CanvasKit にフォールバック
  することがある。`main.dart.wasm` が読み込まれているか Network タブで
  確認しないと、実は動いていなかった、ということが起こりうる

## 背景

Flutter Web のレンダラー選択は、以前は `--web-renderer=html|canvaskit|auto` の
3択だったが、`html` レンダラーは廃止され、現在は実質的に次の2択になっている。

- **デフォルト(CanvasKit)**: JavaScript にコンパイルし、Skia を CanvasKit
  (WebGL経由のWasmビルド) で描画する、これまでの標準的な構成
- **`--wasm`(skwasm)**: アプリ本体を WebAssembly にコンパイルし、新しい
  Skia-in-Wasm レンダラー(skwasm)で描画する。未対応ブラウザには自動で
  JavaScript にフォールバックする

「新しい方が速いはず」という触れ込みだけで終わらせず、実際に手元のアプリと
ブラウザで数値を取って比較してみる。

:::message
本記事ではコードは抜粋のみ掲載する。
:::

## 検証環境

| 項目 | 値 |
|---|---|
| Flutter | 3.44.9 |
| Dart | 3.12.2 |
| 開発環境 | Docker (Debian bookworm-slim ベース、Flutter SDK はバージョン固定) |
| ホストOS | Windows 11 + WSL2 (Ubuntu 24.04) |
| 計測に使ったブラウザ | Microsoft Edge 151.0.4129.72 (64ビット) |
| 計測に使ったマシン | Intel Core i7-1185G7 (第11世代) / RAM 32GB / Intel Iris Xe Graphics(内蔵GPU) |

:::message
Firefox でも試したところ、skwasm ビルド(`--wasm`)が黙って CanvasKit に
フォールバックする現象が発生した。詳細は後述。
:::

## 検証方法

### ベンチ用アプリ

わざと重めの内容を詰めた3画面構成のFlutterアプリを使う。

- **従業員一覧**: 日本語200行の `DataTable`(ソート可能)
- **申請フォーム**: `TextFormField` / `DropdownButtonFormField` /
  `RadioGroup` / `DatePicker` を組み合わせたフォーム
- **ダッシュボード**: KPIカード、`Container` で自前描画した棒グラフ、
  部署別サマリテーブル、直近申請リスト

:::details アプリの全文(参考, lib/main.dart)
```dart
// Flutter Web レンダラ比較ベンチマーク (CanvasKit vs skwasm)
// 目的: 日本語フォント描画・DataTable・フォームのパフォーマンス計測
import 'package:flutter/material.dart';

import 'perf_overlay.dart';

void main() => runApp(const BenchApp());

class BenchApp extends StatelessWidget {
  const BenchApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Web レンダラベンチ',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      // 全画面共通で右下に build/raster 時間の実測オーバーレイを重ねる。
      builder: (context, child) => PerfOverlay(child: child!),
      home: const HomePage(),
    );
  }
}

// ナビゲーション定義 (アイコン, ラベル)
const _navItems = [
  (Icons.list_alt,    '従業員一覧'),
  (Icons.edit_note,   '申請フォーム'),
  (Icons.dashboard,   'ダッシュボード'),
];

// ─── ホーム: レスポンシブナビゲーション ──────────────────────────────────
// 700px 以上 → 固定サイドナビ (NavigationRail)
// 700px 未満 → Drawer (ハンバーガーメニュー)
class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  int _index = 0;

  // 各画面は Scaffold なし — このクラスが Scaffold を一元管理する
  static const _screens = <Widget>[
    ListScreen(),
    FormScreen(),
    DashboardScreen(),
  ];

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(builder: (_, constraints) {
      return constraints.maxWidth >= 700
          ? _wideLayout()
          : _narrowLayout();
    });
  }

  // ── 幅広レイアウト: NavigationRail を左に固定 ──────────────────────────
  Widget _wideLayout() {
    final cs = Theme.of(context).colorScheme;
    return Scaffold(
      body: Row(
        children: [
          // 固定サイドナビ
          NavigationRail(
            extended: true,
            minExtendedWidth: 200,
            selectedIndex: _index,
            onDestinationSelected: (i) => setState(() => _index = i),
            leading: const _SideHeader(),
            destinations: _navItems
                .map((e) => NavigationRailDestination(
                      icon: Icon(e.$1),
                      label: Text(e.$2),
                    ))
                .toList(),
          ),
          const VerticalDivider(width: 1, thickness: 1),
          // コンテンツエリア (AppBar + 画面本体)
          Expanded(
            child: Scaffold(
              appBar: AppBar(
                title: Text(_navItems[_index].$2),
                backgroundColor: cs.inversePrimary,
                automaticallyImplyLeading: false, // ハンバーガー不要
              ),
              body: _screens[_index],
            ),
          ),
        ],
      ),
    );
  }

  // ── 狭いレイアウト: AppBar + Drawer ────────────────────────────────────
  Widget _narrowLayout() {
    final cs = Theme.of(context).colorScheme;
    return Scaffold(
      appBar: AppBar(
        title: Text(_navItems[_index].$2),
        backgroundColor: cs.inversePrimary,
        // Drawer があると Flutter が自動でハンバーガーを表示する
      ),
      drawer: _NavDrawer(
        selectedIndex: _index,
        onSelect: (i) {
          setState(() => _index = i);
          Navigator.pop(context); // Drawer を閉じる
        },
      ),
      body: _screens[_index],
    );
  }
}

// ── サイドナビのヘッダ ──────────────────────────────────────────────────
class _SideHeader extends StatelessWidget {
  const _SideHeader();

  @override
  Widget build(BuildContext context) {
    return const Padding(
      padding: EdgeInsets.fromLTRB(12, 20, 12, 4),
      child: Column(children: [
        Icon(Icons.business_center, size: 36, color: Colors.indigo),
        SizedBox(height: 8),
        Text('業務システム',
            style: TextStyle(fontWeight: FontWeight.bold, fontSize: 14)),
        SizedBox(height: 4),
        Text('Flutter Web ベンチ',
            style: TextStyle(fontSize: 11, color: Colors.black45)),
        SizedBox(height: 12),
        Divider(),
      ]),
    );
  }
}

// ── Drawer ナビゲーション ───────────────────────────────────────────────
class _NavDrawer extends StatelessWidget {
  final int selectedIndex;
  final void Function(int) onSelect;

  const _NavDrawer({required this.selectedIndex, required this.onSelect});

  @override
  Widget build(BuildContext context) {
    return Drawer(
      child: ListView(
        padding: EdgeInsets.zero,
        children: [
          DrawerHeader(
            decoration: BoxDecoration(color: Colors.indigo.shade700),
            child: const Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              mainAxisAlignment: MainAxisAlignment.end,
              children: [
                Icon(Icons.business_center, color: Colors.white, size: 36),
                SizedBox(height: 8),
                Text('業務システム',
                    style: TextStyle(
                        color: Colors.white,
                        fontSize: 18,
                        fontWeight: FontWeight.bold)),
                Text('Flutter Web ベンチ',
                    style: TextStyle(color: Colors.white70, fontSize: 12)),
              ],
            ),
          ),
          ..._navItems.asMap().entries.map((e) {
            final i = e.key;
            final item = e.value;
            return ListTile(
              leading: Icon(item.$1,
                  color: selectedIndex == i ? Colors.indigo : null),
              title: Text(item.$2),
              selected: selectedIndex == i,
              selectedTileColor: Colors.indigo.shade50,
              onTap: () => onSelect(i),
            );
          }),
        ],
      ),
    );
  }
}

// ─── データモデル ─────────────────────────────────────────────────────────
class Employee {
  final String name;
  final String department;
  final String date;
  final int amount;
  final String status;

  const Employee({
    required this.name,
    required this.department,
    required this.date,
    required this.amount,
    required this.status,
  });
}

// ダミーデータ生成 (200行)
List<Employee> generateEmployees() {
  final names = [
    '田中 一郎', '鈴木 花子', '佐藤 健太', '高橋 美咲', '渡辺 直樹',
    '伊藤 さくら', '山田 雄介', '中村 あかり', '小林 翔太', '加藤 麻衣',
    '吉田 大輔', '山口 ゆき', '松本 颯太', '井上 ひなた', '木村 拓也',
    '林 奈々', '清水 悠斗', '山崎 れな', '斎藤 航', '中島 あおい',
  ];
  final departments = [
    '営業部', '開発部', '人事部', '経理部', '総務部',
    '企画部', 'マーケティング部', '法務部', '情報システム部', '広報部',
  ];
  final statuses = ['承認済', '審査中', '差し戻し', '未申請', '完了'];

  return List.generate(200, (i) {
    final year = 2024 + (i % 2);
    final month = (i % 12) + 1;
    final day = (i % 28) + 1;
    return Employee(
      name: names[i % names.length],
      department: departments[i % departments.length],
      date:
          '$year/${month.toString().padLeft(2, '0')}/${day.toString().padLeft(2, '0')}',
      amount: (i + 1) * 12500 + (i % 7) * 3000,
      status: statuses[i % statuses.length],
    );
  });
}

// ─── 画面1: 従業員一覧 (DataTable 200行) ────────────────────────────────
class ListScreen extends StatefulWidget {
  const ListScreen({super.key});

  @override
  State<ListScreen> createState() => _ListScreenState();
}

class _ListScreenState extends State<ListScreen> {
  final _employees = generateEmployees();
  int _sortColumnIndex = 0;
  bool _sortAsc = true;

  Color _statusColor(String status) => switch (status) {
        '承認済' => Colors.green,
        '審査中' => Colors.orange,
        '差し戻し' => Colors.red,
        '完了' => Colors.blue,
        _ => Colors.grey,
      };

  @override
  Widget build(BuildContext context) {
    // LayoutBuilder で利用可能幅を取得し DataTable を引き伸ばす
    // → 幅が余っていればフル幅、列が多ければ横スクロール
    return LayoutBuilder(builder: (context, constraints) {
      return SingleChildScrollView(
        child: SingleChildScrollView(
          scrollDirection: Axis.horizontal,
          child: ConstrainedBox(
            constraints: BoxConstraints(minWidth: constraints.maxWidth),
            child: DataTable(
              sortColumnIndex: _sortColumnIndex,
              sortAscending: _sortAsc,
              columnSpacing: 24,
              headingRowColor:
                  WidgetStateProperty.all(Colors.indigo.shade50),
              columns: [
                DataColumn(
                  label: const Text('氏名',
                      style: TextStyle(fontWeight: FontWeight.bold)),
                  onSort: (i, asc) =>
                      setState(() { _sortColumnIndex = i; _sortAsc = asc; }),
                ),
                DataColumn(
                  label: const Text('部署',
                      style: TextStyle(fontWeight: FontWeight.bold)),
                  onSort: (i, asc) =>
                      setState(() { _sortColumnIndex = i; _sortAsc = asc; }),
                ),
                DataColumn(
                  label: const Text('申請日',
                      style: TextStyle(fontWeight: FontWeight.bold)),
                  onSort: (i, asc) =>
                      setState(() { _sortColumnIndex = i; _sortAsc = asc; }),
                ),
                DataColumn(
                  label: const Text('申請金額',
                      style: TextStyle(fontWeight: FontWeight.bold)),
                  numeric: true,
                  onSort: (i, asc) =>
                      setState(() { _sortColumnIndex = i; _sortAsc = asc; }),
                ),
                DataColumn(
                  label: const Text('ステータス',
                      style: TextStyle(fontWeight: FontWeight.bold)),
                  onSort: (i, asc) =>
                      setState(() { _sortColumnIndex = i; _sortAsc = asc; }),
                ),
              ],
              rows: _employees.map((e) {
                return DataRow(cells: [
                  DataCell(Text(e.name,
                      style: const TextStyle(fontWeight: FontWeight.w500))),
                  DataCell(Text(e.department)),
                  DataCell(Text(e.date)),
                  DataCell(Text(
                    '¥${e.amount.toString().replaceAllMapped(
                          RegExp(r'(\d)(?=(\d{3})+$)'),
                          (m) => '${m[1]},',
                        )}',
                    style: const TextStyle(fontFamily: 'monospace'),
                  )),
                  DataCell(Container(
                    padding: const EdgeInsets.symmetric(
                        horizontal: 10, vertical: 4),
                    decoration: BoxDecoration(
                      color: _statusColor(e.status).withAlpha(30),
                      borderRadius: BorderRadius.circular(12),
                      border: Border.all(
                          color: _statusColor(e.status).withAlpha(100)),
                    ),
                    child: Text(e.status,
                        style: TextStyle(
                            color: _statusColor(e.status), fontSize: 12)),
                  )),
                ]);
              }).toList(),
            ),
          ),
        ),
      );
    });
  }
}

// ─── 画面2: 申請フォーム ────────────────────────────────────────────────
class FormScreen extends StatefulWidget {
  const FormScreen({super.key});

  @override
  State<FormScreen> createState() => _FormScreenState();
}

class _FormScreenState extends State<FormScreen> {
  final _formKey = GlobalKey<FormState>();
  final _nameCtrl = TextEditingController();
  final _amountCtrl = TextEditingController();
  final _reasonCtrl = TextEditingController();

  String? _selectedDept;
  DateTime? _selectedDate;
  String _category = '旅費交通費';
  bool _submitted = false;

  final _departments = [
    '営業部', '開発部', '人事部', '経理部', '総務部',
    '企画部', 'マーケティング部', '法務部', '情報システム部', '広報部',
  ];

  final _categories = [
    '旅費交通費', '接待交際費', '消耗品費', '通信費', '研修費', 'その他',
  ];

  Future<void> _pickDate() async {
    final picked = await showDatePicker(
      context: context,
      initialDate: DateTime.now(),
      firstDate: DateTime(2020),
      lastDate: DateTime(2030),
      helpText: '申請日を選択',
      cancelText: 'キャンセル',
      confirmText: '確定',
    );
    if (picked != null) setState(() => _selectedDate = picked);
  }

  void _submit() {
    if (_formKey.currentState!.validate() &&
        _selectedDate != null &&
        _selectedDept != null) {
      setState(() => _submitted = true);
    } else {
      ScaffoldMessenger.of(context).showSnackBar(const SnackBar(
        content: Text('入力内容を確認してください'),
        backgroundColor: Colors.orange,
      ));
    }
  }

  void _reset() {
    _formKey.currentState!.reset();
    _nameCtrl.clear();
    _amountCtrl.clear();
    _reasonCtrl.clear();
    setState(() {
      _selectedDept = null;
      _selectedDate = null;
      _category = '旅費交通費';
      _submitted = false;
    });
  }

  @override
  void dispose() {
    _nameCtrl.dispose();
    _amountCtrl.dispose();
    _reasonCtrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return _submitted ? _successView() : _formView();
  }

  Widget _successView() {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          const Icon(Icons.check_circle, color: Colors.green, size: 80),
          const SizedBox(height: 16),
          const Text('申請が完了しました',
              style:
                  TextStyle(fontSize: 22, fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          Text('申請者: ${_nameCtrl.text}',
              style: const TextStyle(fontSize: 16)),
          Text('部署: $_selectedDept',
              style: const TextStyle(fontSize: 16)),
          Text('金額: ¥${_amountCtrl.text}',
              style: const TextStyle(fontSize: 16)),
          Text('区分: $_category',
              style: const TextStyle(fontSize: 16)),
          const SizedBox(height: 24),
          ElevatedButton.icon(
            onPressed: _reset,
            icon: const Icon(Icons.refresh),
            label: const Text('新規申請'),
          ),
        ],
      ),
    );
  }

  Widget _formView() {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(24),
      child: Center(
        child: ConstrainedBox(
          constraints: const BoxConstraints(maxWidth: 600),
          child: Form(
            key: _formKey,
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                const Text('基本情報',
                    style: TextStyle(
                        fontSize: 16, fontWeight: FontWeight.bold)),
                const Divider(),
                const SizedBox(height: 8),

                // 申請者氏名
                TextFormField(
                  controller: _nameCtrl,
                  decoration: const InputDecoration(
                    labelText: '申請者氏名 *',
                    hintText: '例: 田中 一郎',
                    prefixIcon: Icon(Icons.person),
                    border: OutlineInputBorder(),
                  ),
                  validator: (v) =>
                      (v == null || v.isEmpty) ? '氏名を入力してください' : null,
                ),
                const SizedBox(height: 16),

                // 所属部署 (Dropdown)
                DropdownButtonFormField<String>(
                  initialValue: _selectedDept,
                  decoration: const InputDecoration(
                    labelText: '所属部署 *',
                    prefixIcon: Icon(Icons.business),
                    border: OutlineInputBorder(),
                  ),
                  hint: const Text('部署を選択してください'),
                  items: _departments
                      .map((d) =>
                          DropdownMenuItem(value: d, child: Text(d)))
                      .toList(),
                  onChanged: (v) => setState(() => _selectedDept = v),
                  validator: (v) =>
                      v == null ? '部署を選択してください' : null,
                ),
                const SizedBox(height: 16),

                // 申請日 (DatePicker)
                InkWell(
                  onTap: _pickDate,
                  child: InputDecorator(
                    decoration: const InputDecoration(
                      labelText: '申請日 *',
                      prefixIcon: Icon(Icons.calendar_today),
                      border: OutlineInputBorder(),
                    ),
                    child: Text(
                      _selectedDate == null
                          ? '日付を選択してください'
                          : '${_selectedDate!.year}年'
                              '${_selectedDate!.month}月'
                              '${_selectedDate!.day}日',
                      style: TextStyle(
                          color: _selectedDate == null
                              ? Colors.grey
                              : Colors.black87),
                    ),
                  ),
                ),
                const SizedBox(height: 24),

                const Text('申請内容',
                    style: TextStyle(
                        fontSize: 16, fontWeight: FontWeight.bold)),
                const Divider(),
                const SizedBox(height: 8),

                // 費用区分 (ラジオボタン)
                // RadioGroup で groupValue/onChanged を一元管理 (3.32 以降の推奨)
                const Text('費用区分 *',
                    style: TextStyle(fontWeight: FontWeight.w500)),
                RadioGroup<String>(
                  groupValue: _category,
                  onChanged: (v) {
                    if (v != null) setState(() => _category = v);
                  },
                  child: Column(
                    children: _categories
                        .map((cat) => RadioListTile<String>(
                              title: Text(cat),
                              value: cat,
                              dense: true,
                            ))
                        .toList(),
                  ),
                ),
                const SizedBox(height: 16),

                // 申請金額
                TextFormField(
                  controller: _amountCtrl,
                  keyboardType: TextInputType.number,
                  decoration: const InputDecoration(
                    labelText: '申請金額 (円) *',
                    hintText: '例: 12500',
                    prefixIcon: Icon(Icons.currency_yen),
                    border: OutlineInputBorder(),
                  ),
                  validator: (v) {
                    if (v == null || v.isEmpty) return '金額を入力してください';
                    if (int.tryParse(v) == null) return '半角数字で入力してください';
                    if (int.parse(v) <= 0) return '0より大きい金額を入力してください';
                    return null;
                  },
                ),
                const SizedBox(height: 16),

                // 申請理由
                TextFormField(
                  controller: _reasonCtrl,
                  maxLines: 3,
                  decoration: const InputDecoration(
                    labelText: '申請理由 *',
                    hintText: '例: 東京出張に伴う交通費および宿泊費',
                    prefixIcon: Icon(Icons.note_alt),
                    border: OutlineInputBorder(),
                    alignLabelWithHint: true,
                  ),
                  validator: (v) =>
                      (v == null || v.isEmpty) ? '申請理由を入力してください' : null,
                ),
                const SizedBox(height: 32),

                // 送信ボタン
                SizedBox(
                  width: double.infinity,
                  height: 48,
                  child: ElevatedButton.icon(
                    onPressed: _submit,
                    icon: const Icon(Icons.send),
                    label: const Text('申請を送信する',
                        style: TextStyle(fontSize: 16)),
                    style: ElevatedButton.styleFrom(
                      backgroundColor: Colors.indigo,
                      foregroundColor: Colors.white,
                    ),
                  ),
                ),
                const SizedBox(height: 12),
                SizedBox(
                  width: double.infinity,
                  height: 48,
                  child: OutlinedButton.icon(
                    onPressed: _reset,
                    icon: const Icon(Icons.clear),
                    label: const Text('入力内容をリセット'),
                  ),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}

// ─── 画面3: ダッシュボード ──────────────────────────────────────────────
class DashboardScreen extends StatelessWidget {
  const DashboardScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text('今月のサマリ',
              style:
                  TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          const SizedBox(height: 12),
          const Wrap(
            spacing: 12,
            runSpacing: 12,
            children: [
              _KpiCard(
                  label: '申請件数',
                  value: '142件',
                  icon: Icons.description,
                  color: Colors.indigo),
              _KpiCard(
                  label: '承認済み',
                  value: '98件',
                  icon: Icons.check_circle,
                  color: Colors.green),
              _KpiCard(
                  label: '審査中',
                  value: '31件',
                  icon: Icons.hourglass_top,
                  color: Colors.orange),
              _KpiCard(
                  label: '差し戻し',
                  value: '13件',
                  icon: Icons.cancel,
                  color: Colors.red),
            ],
          ),
          const SizedBox(height: 24),
          const Text('月別申請金額推移',
              style:
                  TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          const SizedBox(height: 12),
          const _BarChart(),
          const SizedBox(height: 24),
          const Text('部署別申請状況',
              style:
                  TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          const SizedBox(height: 12),
          const _DeptSummaryTable(),
          const SizedBox(height: 24),
          const Text('最近の申請',
              style:
                  TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          const _RecentList(),
        ],
      ),
    );
  }
}

// KPIカード
class _KpiCard extends StatelessWidget {
  final String label;
  final String value;
  final IconData icon;
  final Color color;

  const _KpiCard(
      {required this.label,
      required this.value,
      required this.icon,
      required this.color});

  @override
  Widget build(BuildContext context) {
    return Card(
      elevation: 2,
      child: Container(
        width: 160,
        padding: const EdgeInsets.all(20),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Icon(icon, color: color, size: 32),
            const SizedBox(height: 12),
            Text(value,
                style: TextStyle(
                    fontSize: 28,
                    fontWeight: FontWeight.bold,
                    color: color)),
            const SizedBox(height: 4),
            Text(label,
                style:
                    const TextStyle(color: Colors.black54, fontSize: 13)),
          ],
        ),
      ),
    );
  }
}

// 簡易棒グラフ (外部パッケージ不使用 / Container で描画)
class _BarChart extends StatelessWidget {
  const _BarChart();

  @override
  Widget build(BuildContext context) {
    const data = [
      ('4月', 182), ('5月', 245), ('6月', 198), ('7月', 320),
      ('8月', 275), ('9月', 410), ('10月', 356), ('11月', 289),
      ('12月', 195), ('1月', 230), ('2月', 267), ('3月', 389),
    ];
    const maxVal = 410;

    return Card(
      elevation: 1,
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(children: [
          SizedBox(
            height: 160,
            child: Row(
              crossAxisAlignment: CrossAxisAlignment.end,
              children: data.map((item) {
                final ratio = item.$2 / maxVal;
                return Expanded(
                  child: Padding(
                    padding:
                        const EdgeInsets.symmetric(horizontal: 3),
                    child: Column(
                      mainAxisAlignment: MainAxisAlignment.end,
                      children: [
                        Text('${item.$2}',
                            style: const TextStyle(fontSize: 9)),
                        const SizedBox(height: 2),
                        Container(
                          height: 130 * ratio,
                          decoration: BoxDecoration(
                            color: Colors.indigo.withAlpha(180),
                            borderRadius: const BorderRadius.vertical(
                                top: Radius.circular(4)),
                          ),
                        ),
                      ],
                    ),
                  ),
                );
              }).toList(),
            ),
          ),
          const SizedBox(height: 4),
          Row(
            children: data
                .map((item) => Expanded(
                      child: Text(item.$1,
                          textAlign: TextAlign.center,
                          style: const TextStyle(fontSize: 10)),
                    ))
                .toList(),
          ),
          const SizedBox(height: 4),
          const Text('単位: 万円',
              style:
                  TextStyle(fontSize: 11, color: Colors.black45)),
        ]),
      ),
    );
  }
}

// 部署別サマリテーブル
class _DeptSummaryTable extends StatelessWidget {
  const _DeptSummaryTable();

  @override
  Widget build(BuildContext context) {
    const rows = [
      ('営業部', 32, '¥1,245,000', '89%'),
      ('開発部', 28, '¥987,500', '96%'),
      ('人事部', 15, '¥456,000', '80%'),
      ('経理部', 12, '¥389,000', '100%'),
      ('総務部', 18, '¥567,500', '72%'),
      ('企画部', 9, '¥234,000', '67%'),
      ('マーケティング部', 14, '¥678,000', '93%'),
      ('法務部', 7, '¥198,000', '86%'),
      ('情報システム部', 5, '¥345,000', '100%'),
      ('広報部', 2, '¥89,000', '50%'),
    ];

    return Card(
      elevation: 1,
      child: Padding(
        padding: const EdgeInsets.symmetric(vertical: 8),
        child: Column(children: [
          Container(
            color: Colors.indigo.shade50,
            padding:
                const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
            child: const Row(children: [
              Expanded(
                  flex: 3,
                  child: Text('部署名',
                      style:
                          TextStyle(fontWeight: FontWeight.bold))),
              Expanded(
                  flex: 1,
                  child: Text('件数',
                      textAlign: TextAlign.center,
                      style:
                          TextStyle(fontWeight: FontWeight.bold))),
              Expanded(
                  flex: 2,
                  child: Text('合計金額',
                      textAlign: TextAlign.right,
                      style:
                          TextStyle(fontWeight: FontWeight.bold))),
              Expanded(
                  flex: 1,
                  child: Text('承認率',
                      textAlign: TextAlign.right,
                      style:
                          TextStyle(fontWeight: FontWeight.bold))),
            ]),
          ),
          ...rows.map((r) {
            final rate = int.parse(r.$4.replaceAll('%', ''));
            return Padding(
              padding: const EdgeInsets.symmetric(
                  horizontal: 16, vertical: 10),
              child: Row(children: [
                Expanded(flex: 3, child: Text(r.$1)),
                Expanded(
                    flex: 1,
                    child: Text('${r.$2}件',
                        textAlign: TextAlign.center)),
                Expanded(
                    flex: 2,
                    child: Text(r.$3,
                        textAlign: TextAlign.right,
                        style: const TextStyle(
                            fontFamily: 'monospace'))),
                Expanded(
                  flex: 1,
                  child: Text(r.$4,
                      textAlign: TextAlign.right,
                      style: TextStyle(
                          color: rate >= 90
                              ? Colors.green
                              : Colors.orange,
                          fontWeight: FontWeight.w500)),
                ),
              ]),
            );
          }),
        ]),
      ),
    );
  }
}

// 最近の申請リスト
class _RecentList extends StatelessWidget {
  const _RecentList();

  @override
  Widget build(BuildContext context) {
    const items = [
      ('田中 一郎', '営業部', '旅費交通費', '¥45,000', '審査中', '2024/03/15'),
      ('鈴木 花子', '開発部', '研修費', '¥120,000', '承認済', '2024/03/14'),
      ('佐藤 健太', 'マーケティング部', '接待交際費', '¥78,500', '差し戻し', '2024/03/14'),
      ('高橋 美咲', '人事部', '消耗品費', '¥12,300', '承認済', '2024/03/13'),
      ('渡辺 直樹', '企画部', '通信費', '¥9,800', '完了', '2024/03/13'),
      ('伊藤 さくら', '総務部', 'その他', '¥34,000', '審査中', '2024/03/12'),
      ('山田 雄介', '法務部', '旅費交通費', '¥56,700', '承認済', '2024/03/12'),
      ('中村 あかり', '情報システム部', '消耗品費', '¥24,800', '完了', '2024/03/11'),
      ('小林 翔太', '広報部', '接待交際費', '¥89,500', '審査中', '2024/03/11'),
      ('加藤 麻衣', '経理部', '研修費', '¥65,000', '承認済', '2024/03/10'),
    ];

    final statusColors = {
      '承認済': Colors.green,
      '審査中': Colors.orange,
      '差し戻し': Colors.red,
      '完了': Colors.blue,
      '未申請': Colors.grey,
    };

    return Card(
      elevation: 1,
      child: ListView.separated(
        shrinkWrap: true,
        physics: const NeverScrollableScrollPhysics(),
        itemCount: items.length,
        separatorBuilder: (_, _) => const Divider(height: 1),
        itemBuilder: (context, i) {
          final item = items[i];
          final color = statusColors[item.$5] ?? Colors.grey;
          return ListTile(
            leading: CircleAvatar(
              backgroundColor: Colors.indigo.shade100,
              child: Text(item.$1[0],
                  style:
                      TextStyle(color: Colors.indigo.shade800)),
            ),
            title: Text('${item.$1} (${item.$2})'),
            subtitle: Text('${item.$3} · ${item.$6}'),
            trailing: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              crossAxisAlignment: CrossAxisAlignment.end,
              children: [
                Text(item.$4,
                    style:
                        const TextStyle(fontWeight: FontWeight.bold)),
                Container(
                  padding: const EdgeInsets.symmetric(
                      horizontal: 8, vertical: 2),
                  decoration: BoxDecoration(
                    color: color.withAlpha(30),
                    borderRadius: BorderRadius.circular(8),
                    border:
                        Border.all(color: color.withAlpha(100)),
                  ),
                  child: Text(item.$5,
                      style:
                          TextStyle(color: color, fontSize: 11)),
                ),
              ],
            ),
          );
        },
      ),
    );
  }
}
```
:::

### ビルド

同じソースから、レンダラーだけを変えて2種類ビルドする。

```sh
# CanvasKit(デフォルト)
flutter build web --release -o build/web-canvaskit

# skwasm
flutter build web --release --wasm -o build/web-skwasm
```

### 計測方法

`dart:ui` の `PlatformDispatcher.instance.onReportTimings` を使い、
フレームごとの `buildDuration`(ウィジェットツリー構築)と
`rasterDuration`(実際の描画)を集計するオーバーレイをアプリ自体に
組み込んだ。画面右下に直近120フレームの平均・p90(ミリ秒)とfpsを常時
表示する。

```dart
// lib/perf_overlay.dart (抜粋)
WidgetsBinding.instance.addTimingsCallback((timings) {
  for (final t in timings) {
    buildMs.add(t.buildDuration.inMicroseconds / 1000);
    rasterMs.add(t.rasterDuration.inMicroseconds / 1000);
  }
});
```

(なぜ `PlatformDispatcher.instance.onReportTimings` に直接代入する形に
しなかったかは後述する)

:::details オーバーレイの全文(参考, lib/perf_overlay.dart)
```dart
// フレームごとの build/raster 時間を集計して画面上に表示するオーバーレイ。
// レンダラー(CanvasKit / skwasm)ごとの描画性能を、体感ではなく実測値で
// 比較するための計測用ウィジェット。
import 'dart:collection';
import 'dart:ui';

import 'package:flutter/material.dart';

// ビルド時に --dart-define=BENCH_RENDERER=canvaskit|skwasm を渡すと
// オーバーレイにどちらのビルドか表示される。未指定時は "default"。
const String benchRendererLabel =
    String.fromEnvironment('BENCH_RENDERER', defaultValue: 'default');

class PerfOverlay extends StatefulWidget {
  final Widget child;
  const PerfOverlay({super.key, required this.child});

  @override
  State<PerfOverlay> createState() => _PerfOverlayState();
}

class _PerfOverlayState extends State<PerfOverlay> {
  static const _windowSize = 120;
  final _buildMs = Queue<double>();
  final _rasterMs = Queue<double>();
  int _frameCount = 0;
  bool _expanded = true;

  @override
  void initState() {
    super.initState();
    // PlatformDispatcher.instance.onReportTimings を直接差し替えると、
    // SchedulerBinding が起動時に登録した処理を上書きしてしまい正しく
    // 動作しない。WidgetsBinding(=SchedulerBinding) が提供する複数
    // リスナー対応の addTimingsCallback を使うのが正しい方法。
    WidgetsBinding.instance.addTimingsCallback(_onReportTimings);
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeTimingsCallback(_onReportTimings);
    super.dispose();
  }

  void _onReportTimings(List<FrameTiming> timings) {
    for (final t in timings) {
      _buildMs.add(t.buildDuration.inMicroseconds / 1000);
      _rasterMs.add(t.rasterDuration.inMicroseconds / 1000);
      if (_buildMs.length > _windowSize) _buildMs.removeFirst();
      if (_rasterMs.length > _windowSize) _rasterMs.removeFirst();
      _frameCount++;
    }
    if (mounted) setState(() {});
  }

  double _avg(Queue<double> xs) =>
      xs.isEmpty ? 0 : xs.reduce((a, b) => a + b) / xs.length;

  double _p90(Queue<double> xs) {
    if (xs.isEmpty) return 0;
    final sorted = xs.toList()..sort();
    return sorted[((sorted.length - 1) * 0.9).round()];
  }

  @override
  Widget build(BuildContext context) {
    final avgBuild = _avg(_buildMs);
    final avgRaster = _avg(_rasterMs);
    final p90Build = _p90(_buildMs);
    final p90Raster = _p90(_rasterMs);
    final avgTotal = avgBuild + avgRaster;
    final fps = avgTotal > 0 ? (1000 / avgTotal).clamp(0.0, 999.0) : 0.0;

    return Stack(
      children: [
        widget.child,
        Positioned(
          right: 8,
          bottom: 8,
          child: SafeArea(
            child: Material(
              color: Colors.transparent,
              child: GestureDetector(
                onTap: () => setState(() => _expanded = !_expanded),
                child: Container(
                  padding: const EdgeInsets.symmetric(
                      horizontal: 10, vertical: 8),
                  decoration: BoxDecoration(
                    color: Colors.black.withAlpha(200),
                    borderRadius: BorderRadius.circular(8),
                  ),
                  child: _expanded
                      ? _detail(avgBuild, avgRaster, p90Build, p90Raster, fps)
                      : _badge(fps),
                ),
              ),
            ),
          ),
        ),
      ],
    );
  }

  Widget _badge(double fps) => Text(
        '$benchRendererLabel  ${fps.toStringAsFixed(0)}fps',
        style: const TextStyle(
            color: Colors.white, fontSize: 11, fontFamily: 'monospace'),
      );

  Widget _detail(double avgBuild, double avgRaster, double p90Build,
      double p90Raster, double fps) {
    final style = const TextStyle(
        color: Colors.white70, fontSize: 11, fontFamily: 'monospace');
    final headStyle = const TextStyle(
        color: Colors.white,
        fontSize: 12,
        fontFamily: 'monospace',
        fontWeight: FontWeight.bold);
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      mainAxisSize: MainAxisSize.min,
      children: [
        Text('renderer: $benchRendererLabel', style: headStyle),
        Text('fps(avg)  : ${fps.toStringAsFixed(1)}', style: style),
        Text(
            'build ms  : avg ${avgBuild.toStringAsFixed(2)} / p90 ${p90Build.toStringAsFixed(2)}',
            style: style),
        Text(
            'raster ms : avg ${avgRaster.toStringAsFixed(2)} / p90 ${p90Raster.toStringAsFixed(2)}',
            style: style),
        Text('frames    : $_frameCount (直近${_buildMs.length}件で集計)',
            style: style),
      ],
    );
  }
}
```
:::

DevTools のプロファイラではなく数値をアプリ内に表示させているのは、
CanvasKit版とskwasm版を別タブで並べて同時に見比べられるようにするため。

計測は以下の手順で統一した:

1. ブラウザのキャッシュをクリアした状態でページを開く
2. 従業員一覧画面で同じ量・同じ速さでスクロールする(オーバーレイの
   フレーム数が一定件数たまるまで)
3. 申請フォーム画面に遷移し、同じ内容を入力する
4. ダッシュボード画面に遷移する

## やってみて分かったこと: onReportTimingsは直接上書きしてはいけない

最初は `PlatformDispatcher.instance.onReportTimings = (timings) { ... }`
のようにコールバックを直接代入する形で実装していた。ところがこれで
ビルドして動かすと、オーバーレイの `frames` がスクロールしてもずっと
`0` のまま動かない。

原因は、`PlatformDispatcher.instance.onReportTimings` が単一スロットの
コールバックであり、Flutter フレームワーク自身(`SchedulerBinding`、
`WidgetsFlutterBinding` の初期化時)が既にそこへ自分のハンドラーを
登録済みだったこと。アプリ側の `initState()` で後から直接代入すると、
フレームワーク側の登録を黙って上書きしてしまい、しかもエラーも出ない
ため気づきにくい。

修正は、複数リスナーに対応した公式API `WidgetsBinding.instance
.addTimingsCallback()` / `removeTimingsCallback()` を使うこと。これなら
既存の登録を壊さずにフレーム計測を追加できる。ヘッドレスブラウザで
スクロールを行うテストを書いて再現・確認した(`frames: 0` →
`frames: 64` に回復)。

## やってみて分かったこと: ビルド成果物の中身

数値の話に入る前に、ビルド出力を覗いていて気づいたことを書いておく。

`build/web-canvaskit` と `build/web-skwasm` を比較すると:

```
build/web-canvaskit/
├── main.dart.js       2.68MB
└── canvaskit/         37MB   ← CanvasKit本体一式

build/web-skwasm/
├── main.dart.js       2.68MB (フォールバック用)
├── main.dart.wasm     2.28MB
├── main.dart.mjs      32KB
└── canvaskit/         37MB   ← こちらにも同じものが入っている
```

`--wasm` ビルドでも `canvaskit/` ディレクトリは(バイト単位で同一の内容が)
そのまま含まれていた。さらにややこしいことに、CanvasKit本体のバンドルの
中には `skwasm.wasm` / `skwasm_heavy.wasm` というファイルもある。これは
「CanvasKit自身のマルチスレッド版」であり、Flutter側の `--wasm` レンダラー
(=今回比較しているskwasm)とは別物で、名前が衝突していて紛らわしい。

つまり **ビルドディレクトリの合計サイズをそのまま比較するのはミスリーディング**。
実際にブラウザが初回ロード時にダウンロードするのは、ブラウザの機能検出に
応じて動的に選ばれる一部だけなので、静的なファイルサイズではなく
DevTools の Network タブで実際に転送された量を見る必要がある。

## やってみて分かったこと: skwasmが黙ってフォールバックすることがある

`--wasm` でビルドしたページを Firefox で開き、Network タブで確認したところ
気づいた。

- [CanvasKit版(:8091, Firefox)](https://gyazo.com/5f5d617e6238558acdc9c2b35945ce24)
- [skwasm版のつもりが…(:8092, Firefox)](https://gyazo.com/d34c21162043cb963e4ad48d5fc6e415)

skwasm版のはずの `:8092` で読み込まれていたのは `main.dart.wasm` ではなく
`main.dart.js`、しかも `www.gstatic.com` から `canvaskit.js` /
`canvaskit.wasm` まで取得していた。リクエスト数(39件)・転送量
(10.43MB / 5.44MB)も CanvasKit版とほぼ一致しており、**skwasm ビルドが
Firefox 上では黙って CanvasKit にフォールバックしていた**ことが分かる。

「未対応ブラウザには自動でJSにフォールバックする」という仕様通りではある
ものの、Network タブで `main.dart.wasm` が読み込まれているかを確認しない
限り、実際にskwasmが動いているのかどうかは分からない。この後の計測は
すべて、`main.dart.wasm` の読み込みを確認できた **Microsoft Edge** で
行っている。

## 計測結果

### 1. 初回ロード(キャッシュなし、Microsoft Edge)

| | CanvasKit (:8091) | skwasm (:8092) |
|---|---|---|
| リクエスト数 | 40 | 43 |
| 転送量 | 4.8 MB | **4.0 MB** |
| 展開後サイズ | 9.0 MB | **6.5 MB** |
| Finish(全リクエスト完了) | 1.19 s | **795 ms** |
| DOMContentLoaded | 52 ms | 52 ms |
| Load | 52 ms | 53 ms |

![CanvasKit版のNetworkタブ](https://i.gyazo.com/ceb9b1350369ab1bcd7d011b3b3f6972.png)
*CanvasKit版のNetworkタブ*

![skwasm版のNetworkタブ](https://i.gyazo.com/f11e4f1732e91578ffcabaa10fdef2fd.png)
*skwasm版のNetworkタブ*

DOMContentLoaded / Load はどちらもほぼ同じ(52ms前後)で、これは初期HTMLの
パース完了タイミングであり、アプリ本体(JS/Wasm)のダウンロード量とは
あまり関係がないことが分かる。実際の重さの違いは Finish に表れていて、
skwasm 版の方が明確に速い。中身を見ると理由は単純で、
`main.dart.wasm`(2.3MB)+ `skwasm.wasm`(1.2MB)の合計が、
`main.dart.js`(2.7MB)+ `canvaskit.wasm`(1.7MB)+ `canvaskit.js` より
軽い。

### 2. 従業員一覧(200行DataTable)スクロール時

| | CanvasKit | skwasm |
|---|---|---|
| fps (avg) | **409.6** | 305.0 |
| build ms (avg / p90) | 1.47 / 3.25 | **0.75 / 2.04** |
| raster ms (avg / p90) | **0.97 / 1.31** | 2.53 / 3.70 |

![CanvasKit版のオーバーレイ](https://i.gyazo.com/4af23a4341bb19dd28d355fd7a240e64.png)
*CanvasKit版のオーバーレイ*

![skwasm版のオーバーレイ](https://i.gyazo.com/dbeec9c9b553bde1f014f7f1719d9b2f.png)
*skwasm版のオーバーレイ*

興味深いのは、**build(ウィジェットツリー構築)は skwasm の方が速いのに、
raster(実際のピクセル描画)は CanvasKit の方が速く、合計の fps では
CanvasKit が上回っている**こと。「新しいレンダラーの方が何でも速い」という
単純な話にはならず、処理の種類によって得意不得意が分かれるようだ。

### 3. 申請フォーム入力時

| | CanvasKit | skwasm |
|---|---|---|
| fps (avg) | 196.2 | **262.2** |
| build ms (avg / p90) | 4.15 / 7.74 | **2.07 / 3.46** |
| raster ms (avg / p90) | **0.95 / 1.40** | 1.75 / 2.40 |

![CanvasKit版のオーバーレイ](https://i.gyazo.com/a773e0d7728b122a1693ebf56b8d9fe5.png)
*CanvasKit版のオーバーレイ*

![skwasm版のオーバーレイ](https://i.gyazo.com/b81a2377c97837a818751afff2691048.png)
*skwasm版のオーバーレイ*

スクロール時とは逆の結果になった。フォーム入力は打鍵のたびに
`setState` が走ってウィジェットツリーを再構築する、build寄りの負荷が
中心のシナリオで、build性能で優る skwasm が総合fpsでも上回っている。

ここまでの2つの計測から傾向が見えてくる: **raster性能は常にCanvasKitが
優位、build性能は常にskwasmが優位**。どちらの構成が有利かは、アプリの
処理がbuild寄り(頻繁な再描画・状態更新)かraster寄り(大量の要素を
一度に描画するスクロールなど)かによって変わるようだ。

### 4. 画面遷移時(従業員一覧→申請フォーム→ダッシュボードを2周)

| | CanvasKit | skwasm |
|---|---|---|
| fps (avg) | **217.5** | 191.9 |
| build ms (avg / p90) | 3.55 / 5.74 | **2.85 / 2.33** |
| raster ms (avg / p90) | **1.04 / 1.46** | 2.36 / 2.88 |

![CanvasKit版のオーバーレイ](https://i.gyazo.com/f019628d4474839e81173c2c74ebb0de.png)
*CanvasKit版のオーバーレイ*

![skwasm版のオーバーレイ](https://i.gyazo.com/1202b5f8e384aeae9603a83d2d5bb6ca.png)
*skwasm版のオーバーレイ*

build ms は今回も skwasm が速いが、raster ms の差が大きく、総合fpsは
CanvasKit が上回った。画面遷移は新しい画面(KPIカード・棒グラフ・
テーブルなど)を丸ごと描き直す raster 寄りの負荷なので、スクロール時と
同じ傾向になったのは筋が通っている。

## 考察

### 初回ロード: skwasmの方が軽くて速い

転送量(4.0MB vs 4.8MB)・展開後サイズ(6.5MB vs 9.0MB)・全リクエスト
完了までの時間(795ms vs 1.19s)、いずれも skwasm が上回った。
`main.dart.wasm`(2.3MB)+ `skwasm.wasm`(1.2MB)の合計が、
`main.dart.js`(2.7MB)+ `canvaskit.wasm`(1.7MB)+ `canvaskit.js` より
軽いという、素直な理由による差だと考えられる。

### ランタイム性能: 「build寄り」か「raster寄り」かで勝敗が変わる

3つのシナリオの結果を並べると、はっきりしたパターンが見える。

| シナリオ | 負荷の性質 | build ms(avg) | raster ms(avg) | fps(avg)の勝者 |
|---|---|---|---|---|
| DataTableスクロール | raster寄り | skwasm 0.75 < CanvasKit 1.47 | CanvasKit 0.97 < skwasm 2.53 | **CanvasKit**(409.6 vs 305.0) |
| フォーム入力 | build寄り | skwasm 2.07 < CanvasKit 4.15 | CanvasKit 0.95 < skwasm 1.75 | **skwasm**(262.2 vs 196.2) |
| 画面遷移 x2 | raster寄り | skwasm 2.85 < CanvasKit 3.55 | CanvasKit 1.04 < skwasm 2.36 | **CanvasKit**(217.5 vs 191.9) |

**build性能は3シナリオすべてでskwasmが上回り、raster性能は3シナリオ
すべてでCanvasKitが上回った。** その上で総合fpsは、build時間の割合が
大きいシナリオ(打鍵のたびに再描画が走るフォーム)ではskwasmが、
raster時間の割合が大きいシナリオ(大量の要素を一度に描き直す
スクロールや画面遷移)ではCanvasKitが勝つ、という一貫した結果になった。

これは「新しいレンダラーの方が何でも速い」という単純な話ではなく、
**アプリの中で頻繁に発生する処理の性質によって向き不向きが変わる**
ことを示していると思う。今回のベンチアプリでいえば:

- 大きな `DataTable` やダッシュボードのような、静的な内容を一気に
  大量描画する画面が中心のアプリ → CanvasKit が有利な可能性
- フォームや入力補完のように、状態変化のたびに細かく再描画が走る
  アプリ → skwasm が有利な可能性

ただしこれはあくまで今回の1アプリ・1マシン・1ブラウザでの結果であり、
一般化するには複数のアプリ・環境での追試が必要だと思う。

## まとめと今回の限界

- 計測は1台のマシン・1ブラウザでのみ行っている(複数環境での再現性は未検証)
- GPUは内蔵(Intel Iris Xe)。discrete GPU環境では raster 側の結果が
  変わる可能性がある
- 各シナリオ1回ずつの計測であり、複数回試行した上での平均・分散は
  取っていない
- 手動でのスクロール・入力操作による計測のため、操作量・速度を厳密に
  そろえられているわけではない(オーバーレイの直近120フレームでの
  集計値を使うことである程度は緩和している)

なお、ビルド・計測環境は Docker + `Makefile` で完結するようにしてあり、
`make bench` の一発で両レンダラーをリリースビルドして別ポートで同時配信
できる状態にしてある(コードは非公開のため割愛)。
