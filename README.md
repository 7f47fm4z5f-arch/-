git init
git add .
git commit -m "最初のコミット：ひよこ先生プロジェクト開始"
git remote add origin https://github.com/あなたのユーザー名/hiyoko-sensei.git
git branch -M main
git push -u origin main
import 'package:flutter/material.dart';
import 'package:hive/hive.dart';
import 'package:path_provider/path_provider.dart';
import 'dart:async';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final dir = await getApplicationDocumentsDirectory();
  Hive.init(dir.path);
  await Hive.openBox('records');
  await Hive.openBox('trash');
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'ひよこ先生',
      home: RecordPage(),
    );
  }
}

class RecordPage extends StatefulWidget {
  @override
  _RecordPageState createState() => _RecordPageState();
}

class _RecordPageState extends State<RecordPage> {
  final recordBox = Hive.box('records');
  final trashBox = Hive.box('trash');
  bool autoSave = true; // ⚙️設定で切り替え可能

  void _saveRecord(String content) async {
    final now = DateTime.now();
    final record = {
      'content': content,
      'timestamp': now.toIso8601String(),
      'deleted': false
    };

    if (autoSave) {
      recordBox.add(record);
    } else {
      _confirmSave(record);
    }
  }

  void _confirmSave(Map record) async {
    final shouldSave = await showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text('保存しますか？'),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context, false), child: Text('捨てる')),
          TextButton(onPressed: () => Navigator.pop(context, true), child: Text('保存')),
        ],
      ),
    );

    if (shouldSave == true) recordBox.add(record);
  }

  void _moveToTrash(int index) {
    final item = recordBox.getAt(index);
    item['deleted'] = true;
    trashBox.add(item);
    recordBox.deleteAt(index);
  }

  void _restoreFromTrash(int index) {
    final item = trashBox.getAt(index);
    item['deleted'] = false;
    recordBox.add(item);
    trashBox.deleteAt(index);
  }

  void _autoDeleteOldTrash() {
    final now = DateTime.now();
    final toDelete = <int>[];

    for (int i = 0; i < trashBox.length; i++) {
      final item = trashBox.getAt(i);
      final time = DateTime.parse(item['timestamp']);
      if (now.difference(time).inDays > 30) {
        toDelete.add(i);
      }
    }

    for (final i in toDelete.reversed) {
      trashBox.deleteAt(i);
    }
  }

  @override
  Widget build(BuildContext context) {
    final records = recordBox.values.toList();
    return Scaffold(
      appBar: AppBar(title: Text('ひよこ先生📒')),
      body: ListView.builder(
        itemCount: records.length,
        itemBuilder: (context, index) {
          final item = records[index];
          return ListTile(
            title: Text(item['content']),
            subtitle: Text(item['timestamp']),
            onLongPress: () => _moveToTrash(index),
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _saveRecord("今日の学習記録✨"),
        child: Icon(Icons.add),
      ),
    );
  }
}
// App.tsx
import React, { useState } from 'react';
import { StyleSheet, View, TextInput, Text, TouchableOpacity, FlatList, Alert } from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';

export default function App() {
  const [memo, setMemo] = useState('');
  const [memos, setMemos] = useState<string[]>([]);

  // 🔹メモを保存
  const saveMemo = async () => {
    if (!memo.trim()) return;
    const updated = [...memos, memo];
    setMemos(updated);
    await AsyncStorage.setItem('memos', JSON.stringify(updated));
    setMemo('');
  };

  // 🔹保存データを読み込み
  const loadMemos = async () => {
    const data = await AsyncStorage.getItem('memos');
    if (data) setMemos(JSON.parse(data));
  };

  React.useEffect(() => {
    loadMemos();
  }, []);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>📝 メモアプリ</Text>

      <TextInput
        style={styles.input}
        placeholder="メモを入力..."
        value={memo}
        onChangeText={setMemo}
      />

      <TouchableOpacity style={styles.button} onPress={saveMemo}>
        <Text style={styles.buttonText}>保存</Text>
      </TouchableOpacity>

      <FlatList
        data={memos}
        keyExtractor={(item, index) => index.toString()}
        renderItem={({ item }) => <Text style={styles.memoItem}>{item}</Text>}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20, backgroundColor: '#fff' },
  title: { fontSize: 24, fontWeight: 'bold', textAlign: 'center', marginVertical: 20 },
  input: { borderWidth: 1, borderColor: '#ccc', borderRadius: 8, padding: 10, marginBottom: 10 },
  button: { backgroundColor: '#007bff', padding: 10, borderRadius: 8, alignItems: 'center' },
  buttonText: { color: '#fff', fontWeight: 'bold' },
  memoItem: { backgroundColor: '#f9f9f9', padding: 10, marginVertical: 5, borderRadius: 8 },
});
// App.tsx
import React, { useState, useEffect } from 'react';
import { StyleSheet, View, TextInput, Text, TouchableOpacity, FlatList, Alert } from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';

type MemoItem = {
  id: string;
  text: string;
  deleted: boolean;
  deletedAt?: number;
};

export default function App() {
  const [memo, setMemo] = useState('');
  const [memos, setMemos] = useState<MemoItem[]>([]);
  const [showTrash, setShowTrash] = useState(false);

  // 🔹メモを保存（通常 or 捨てる）
  const saveMemo = async (discard = false) => {
    if (!memo.trim()) return;
    const newMemo: MemoItem = {
      id: Date.now().toString(),
      text: memo,
      deleted: discard,
      deletedAt: discard ? Date.now() : undefined,
    };
    const updated = [...memos, newMemo];
    setMemos(updated);
    await AsyncStorage.setItem('memos', JSON.stringify(updated));
    setMemo('');
  };

  // 🔹30日経過したメモを削除
  const cleanupTrash = async () => {
    const now = Date.now();
    const filtered = memos.filter(m => !(m.deleted && m.deletedAt && now - m.deletedAt > 30 * 24 * 60 * 60 * 1000));
    if (filtered.length !== memos.length) {
      setMemos(filtered);
      await AsyncStorage.setItem('memos', JSON.stringify(filtered));
    }
  };

  // 🔹ゴミ箱から復元
  const restoreMemo = async (id: string) => {
    const updated = memos.map(m => (m.id === id ? { ...m, deleted: false, deletedAt: undefined } : m));
    setMemos(updated);
    await AsyncStorage.setItem('memos', JSON.stringify(updated));
  };

  // 🔹初期ロード＋定期クリーンアップ
  useEffect(() => {
    (async () => {
      const data = await AsyncStorage.getItem('memos');
      if (data) setMemos(JSON.parse(data));
    })();
    cleanupTrash();
  }, []);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>{showTrash ? '🗑 ゴミ箱' : '📝 メモ一覧'}</Text>

      {!showTrash && (
        <>
          <TextInput
            style={styles.input}
            placeholder="メモを入力..."
            value={memo}
            onChangeText={setMemo}
          />
          <View style={styles.buttonRow}>
            <TouchableOpacity style={styles.saveButton} onPress={() => saveMemo(false)}>
              <Text style={styles.buttonText}>自動保存</Text>
            </TouchableOpacity>
            <TouchableOpacity style={styles.trashButton} onPress={() => saveMemo(true)}>
              <Text style={styles.buttonText}>捨てて保存</Text>
            </TouchableOpacity>
          </View>
        </>
      )}

      <FlatList
        data={memos.filter(m => m.deleted === showTrash)}
        keyExtractor={item => item.id}
        renderItem={({ item }) => (
          <TouchableOpacity
            style={styles.memoItem}
            onPress={() => {
              if (showTrash) restoreMemo(item.id);
            }}>
            <Text style={styles.memoText}>{item.text}</Text>
            {showTrash && <Text style={styles.restoreHint}>（タップで復元）</Text>}
          </TouchableOpacity>
        )}
      />

      <TouchableOpacity style={styles.switchButton} onPress={() => setShowTrash(!showTrash)}>
        <Text style={styles.switchText}>{showTrash ? '📄 メモ一覧へ戻る' : '🗑 ゴミ箱を見る'}</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20, backgroundColor: '#fff' },
  title: { fontSize: 22, fontWeight: 'bold', textAlign: 'center', marginVertical: 20 },
  input: { borderWidth: 1, borderColor: '#ccc', borderRadius: 8, padding: 10, marginBottom: 10 },
  buttonRow: { flexDirection: 'row', justifyContent: 'space-around', marginBottom: 20 },
  saveButton: { backgroundColor: '#007bff', padding: 10, borderRadius: 8, width: '45%', alignItems: 'center' },
  trashButton: { backgroundColor: '#dc3545', padding: 10, borderRadius: 8, width: '45%', alignItems: 'center' },
  buttonText: { color: '#fff', fontWeight: 'bold' },
  memoItem: { backgroundColor: '#f9f9f9', padding: 10, marginVertical: 5, borderRadius: 8 },
  memoText: { fontSize: 16 },
  restoreHint: { fontSize: 12, color: '#555' },
  switchButton: { marginTop: 20, alignItems: 'center' },
  switchText: { color: '#007bff', fontSize: 16 },
});
