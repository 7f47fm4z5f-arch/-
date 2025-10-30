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
