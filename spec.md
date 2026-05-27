<!DOCTYPE html>

<html lang="ja">

<head>

&#x20; <meta charset="UTF-8" />

&#x20; <title>講義文字起こし＆要約デモ</title>

&#x20; <style>

&#x20;   body { font-family: Arial, sans-serif; padding: 20px; }

&#x20;   .section { margin-bottom: 20px; }

&#x20;   .output { border: 1px solid #ccc; padding: 12px; border-radius: 6px; min-height: 80px; white-space: pre-wrap; }

&#x20;   button { margin-top: 6px; }

&#x20; </style>

</head>

<body>

&#x20; <h1>講義文字起こし＆要約デモ</h1>



&#x20; <div class="section">

&#x20;   <label>講義名（任意）</label><br/>

&#x20;   <input type="text" id="lectureName" placeholder="講義名を入力" />

&#x20; </div>



&#x20; <div class="section">

&#x20;   <label>音声ファイルを選択</label><br/>

&#x20;   <input type="file" id="audioFile" accept=".wav,.mp3,.m4a,.flac" />

&#x20; </div>



&#x20; <div class="section">

&#x20;   <button id="runBtn">実行</button>

&#x20; </div>



&#x20; <div class="section">

&#x20;   <h3>文字起こし</h3>

&#x20;   <div id="transcription" class="output"></div>

&#x20;   <button id="copyTranscriptBtn" style="display:none;">コピー</button>

&#x20; </div>



&#x20; <div class="section">

&#x20;   <h3>要約</h3>

&#x20;   <div id="summary" class="output"></div>

&#x20;   <button id="copySummaryBtn" style="display:none;">コピー</button>

&#x20; </div>



&#x20; <div class="section">

&#x20;   <h3>要点</h3>

&#x20;   <ul id="keyPoints"></ul>

&#x20; </div>



&#x20; <div class="section" id="errorArea" style="color: red;"></div>



&#x20; <script>

&#x20;   const runBtn = document.getElementById('runBtn');

&#x20;   const audioFileInput = document.getElementById('audioFile');

&#x20;   const transcriptionDiv = document.getElementById('transcription');

&#x20;   const summaryDiv = document.getElementById('summary');

&#x20;   const keyPointsUl = document.getElementById('keyPoints');

&#x20;   const errorArea = document.getElementById('errorArea');

&#x20;   const copyTranscriptBtn = document.getElementById('copyTranscriptBtn');

&#x20;   const copySummaryBtn = document.getElementById('copySummaryBtn');



&#x20;   function resetOutputs() {

&#x20;     transcriptionDiv.textContent = '';

&#x20;     summaryDiv.textContent = '';

&#x20;     keyPointsUl.innerHTML = '';

&#x20;     errorArea.textContent = '';

&#x20;     copyTranscriptBtn.style.display = 'none';

&#x20;     copySummaryBtn.style.display = 'none';

&#x20;   }



&#x20;   async function copyToClipboard(text) {

&#x20;     try {

&#x20;       await navigator.clipboard.writeText(text);

&#x20;       alert('コピーしました');

&#x20;     } catch (e) {

&#x20;       alert('コピーに失敗しました');

&#x20;     }

&#x20;   }



&#x20;   copyTranscriptBtn.addEventListener('click', () => copyToClipboard(transcriptionDiv.textContent));

&#x20;   copySummaryBtn.addEventListener('click', () => copyToClipboard(summaryDiv.textContent));



&#x20;   runBtn.addEventListener('click', async () => {

&#x20;     resetOutputs();

&#x20;     const lectureName = document.getElementById('lectureName').value;

&#x20;     const file = audioFileInput.files\[0];



&#x20;     if (!file) {

&#x20;       errorArea.textContent = '音声ファイルを選択してください。';

&#x20;       return;

&#x20;     }



&#x20;     const formData = new FormData();

&#x20;     formData.append('lectureName', lectureName);

&#x20;     formData.append('audioFile', file);



&#x20;     try {

&#x20;       const res = await fetch('/process', {

&#x20;         method: 'POST',

&#x20;         body: formData

&#x20;       });



&#x20;       if (!res.ok) {

&#x20;         const err = await res.json().catch(() => ({}));

&#x20;         const msg = err.message || 'Error while processing.';

&#x20;         errorArea.textContent = msg;

&#x20;         return;

&#x20;       }



&#x20;       const payload = await res.json();

&#x20;       transcriptionDiv.textContent = payload.transcription || '';

&#x20;       summaryDiv.textContent = payload.summary || '';

&#x20;       // キーポイントを表示

&#x20;       keyPointsUl.innerHTML = '';

&#x20;       (payload.keyPoints || \[]).forEach((kp) => {

&#x20;         const li = document.createElement('li');

&#x20;         li.textContent = kp;

&#x20;         keyPointsUl.appendChild(li);

&#x20;       });



&#x20;       if (payload.transcription) copyTranscriptBtn.style.display = 'inline-block';

&#x20;       if (payload.summary) copySummaryBtn.style.display = 'inline-block';

&#x20;     } catch (err) {

&#x20;       errorArea.textContent = 'API呼び出し中にエラーが発生しました。';

&#x20;     }

&#x20;   });

&#x20; </script>

</body>

</html>



// server.js

const express = require('express');

const multer = require('multer');

const fetch = require('node-fetch');

const fs = require('fs');

const path = require('path');

const {SpeechClient} = require('@google-cloud/speech'); // Google Cloud Speech-to-Text

require('dotenv').config();



const app = express();

const upload = multer({ dest: 'uploads/' });



// 簡易的なパス

app.post('/process', upload.single('audioFile'), async (req, res) => {

&#x20; try {

&#x20;   const lectureName = req.body.lectureName || '';

&#x20;   const file = req.file;



&#x20;   if (!file) {

&#x20;     return res.status(400).json({ message: '音声ファイルを選択してください。' });

&#x20;   }



&#x20;   // 1) 音声ファイルを文字起こし（Speech-to-Text の例）

&#x20;   const transcription = await transcribeAudio(file.path);



&#x20;   // 2) Gemini API で要約

&#x20;   const summary = await summarizeWithGemini(transcription);



&#x20;   // 要点（仮の抽出。実際には要約から抽出するか、別APIで抽出）

&#x20;   const keyPoints = extractKeyPoints(summary);



&#x20;   // 一時ファイル削除

&#x20;   fs.unlinkSync(file.path);



&#x20;   res.json({

&#x20;     transcription,

&#x20;     summary,

&#x20;     keyPoints

&#x20;   });

&#x20; } catch (err) {

&#x20;   console.error(err);

&#x20;   res.status(500).json({ message: '内部エラーが発生しました。' });

&#x20; }

});



// 文字起こし関数（Google Cloud Speech-to-Text の例）

async function transcribeAudio(filePath) {

&#x20; const client = new SpeechClient();

&#x20; const audioBytes = fs.readFileSync(filePath).toString('base64');

&#x20; const request = {

&#x20;   audio: { content: audioBytes },

&#x20;   config: {

&#x20;     encoding: 'LINEAR16', // 録音フォーマットに合わせる

&#x20;     languageCode: 'ja-JP'

&#x20;   }

&#x20; };

&#x20; const \[response] = await client.recognize(request);

&#x20; const transcription = (response.results || \[])

&#x20;   .map(res => res.alternatives\[0].transcript)

&#x20;   .join('\\n');

&#x20; return transcription;

}



// Gemini API で要約する関数（モデル・エンドポイントは公式ドキュメントを参照）

async function summarizeWithGemini(text) {

&#x20; const apiKey = process.env.GEMINI\_API\_KEY;

&#x20; const model = process.env.GEMINI\_MODEL || 'models/your-gemini-model';

&#x20; if (!apiKey) throw new Error('Gemini API Key not configured.');



&#x20; const prompt = `以下の講義の文字起こしを要約してください：\\n\\n${text}`;



&#x20; // 実際のエンドポイントは公式ドキュメントを参照してください

&#x20; const endpoint = `https://gemini.googleapis.com/v1/models/${encodeURIComponent(model)}:predict`;



&#x20; const body = {

&#x20;   prompt: { text: prompt }, // 仕様はモデルによって異なる

&#x20;   max\_tokens: 200, // 例

&#x20; };



&#x20; const resp = await fetch(endpoint, {

&#x20;   method: 'POST',

&#x20;   headers: {

&#x20;     'Authorization': `Bearer ${apiKey}`,

&#x20;     'Content-Type': 'application/json'

&#x20;   },

&#x20;   body: JSON.stringify(body)

&#x20; });



&#x20; if (!resp.ok) {

&#x20;   const err = await resp.text();

&#x20;   throw new Error(`Gemini API error: ${err}`);

&#x20; }



&#x20; const data = await resp.json();

&#x20; // 返却形式はモデルにより異なる。以下は例です

&#x20; const text = data?.choices?.\[0]?.text || data?.result?.summary || '';

&#x20; return text.trim();

}



// 要点抽出（要約から3〜5個の箇条書き風に分割する簡易処理）

function extractKeyPoints(summary) {

&#x20; // 簡易実装: 各文を改行または句点で分割して3〜5個を抽出

&#x20; if (!summary) return \[];

&#x20; const sentences = summary.split(/(?<=\[。．！!？])\\s+/);

&#x20; const points = sentences.slice(0, 5).map(s => s.trim()).filter(Boolean);

&#x20; return points;

}



const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {

&#x20; console.log(`Server running on port ${PORT}`);

});

