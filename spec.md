\<\!DOCTYPE html\>

\<html lang="ja"\>

\<head\>

  \<meta charset="UTF-8" /\>

  \<title\>講義文字起こし＆要約デモ\</title\>

  \<style\>

    body { font-family: Arial, sans-serif; padding: 20px; }

    .section { margin-bottom: 20px; }

    .output { border: 1px solid \#ccc; padding: 12px; border-radius: 6px; min-height: 80px; white-space: pre-wrap; }

    button { margin-top: 6px; }

  \</style\>

\</head\>

\<body\>

  \<h1\>講義文字起こし＆要約デモ\</h1\>

  \<div class="section"\>

    \<label\>講義名（任意）\</label\>\<br/\>

    \<input type="text" id="lectureName" placeholder="講義名を入力" /\>

  \</div\>

  \<div class="section"\>

    \<label\>音声ファイルを選択\</label\>\<br/\>

    \<input type="file" id="audioFile" accept=".wav,.mp3,.m4a,.flac" /\>

  \</div\>

  \<div class="section"\>

    \<button id="runBtn"\>実行\</button\>

  \</div\>

  \<div class="section"\>

    \<h3\>文字起こし\</h3\>

    \<div id="transcription" class="output"\>\</div\>

    \<button id="copyTranscriptBtn" style="display:none;"\>コピー\</button\>

  \</div\>

  \<div class="section"\>

    \<h3\>要約\</h3\>

    \<div id="summary" class="output"\>\</div\>

    \<button id="copySummaryBtn" style="display:none;"\>コピー\</button\>

  \</div\>

  \<div class="section"\>

    \<h3\>要点\</h3\>

    \<ul id="keyPoints"\>\</ul\>

  \</div\>

  \<div class="section" id="errorArea" style="color: red;"\>\</div\>

  \<script\>

    const runBtn \= document.getElementById('runBtn');

    const audioFileInput \= document.getElementById('audioFile');

    const transcriptionDiv \= document.getElementById('transcription');

    const summaryDiv \= document.getElementById('summary');

    const keyPointsUl \= document.getElementById('keyPoints');

    const errorArea \= document.getElementById('errorArea');

    const copyTranscriptBtn \= document.getElementById('copyTranscriptBtn');

    const copySummaryBtn \= document.getElementById('copySummaryBtn');

    function resetOutputs() {

      transcriptionDiv.textContent \= '';

      summaryDiv.textContent \= '';

      keyPointsUl.innerHTML \= '';

      errorArea.textContent \= '';

      copyTranscriptBtn.style.display \= 'none';

      copySummaryBtn.style.display \= 'none';

    }

    async function copyToClipboard(text) {

      try {

        await navigator.clipboard.writeText(text);

        alert('コピーしました');

      } catch (e) {

        alert('コピーに失敗しました');

      }

    }

    copyTranscriptBtn.addEventListener('click', () \=\> copyToClipboard(transcriptionDiv.textContent));

    copySummaryBtn.addEventListener('click', () \=\> copyToClipboard(summaryDiv.textContent));

    runBtn.addEventListener('click', async () \=\> {

      resetOutputs();

      const lectureName \= document.getElementById('lectureName').value;

      const file \= audioFileInput.files\[0\];

      if (\!file) {

        errorArea.textContent \= '音声ファイルを選択してください。';

        return;

      }

      const formData \= new FormData();

      formData.append('lectureName', lectureName);

      formData.append('audioFile', file);

      try {

        const res \= await fetch('/process', {

          method: 'POST',

          body: formData

        });

        if (\!res.ok) {

          const err \= await res.json().catch(() \=\> ({}));

          const msg \= err.message || 'Error while processing.';

          errorArea.textContent \= msg;

          return;

        }

        const payload \= await res.json();

        transcriptionDiv.textContent \= payload.transcription || '';

        summaryDiv.textContent \= payload.summary || '';

        // キーポイントを表示

        keyPointsUl.innerHTML \= '';

        (payload.keyPoints || \[\]).forEach((kp) \=\> {

          const li \= document.createElement('li');

          li.textContent \= kp;

          keyPointsUl.appendChild(li);

        });

        if (payload.transcription) copyTranscriptBtn.style.display \= 'inline-block';

        if (payload.summary) copySummaryBtn.style.display \= 'inline-block';

      } catch (err) {

        errorArea.textContent \= 'API呼び出し中にエラーが発生しました。';

      }

    });

  \</script\>

\</body\>

\</html\>

// server.js

const express \= require('express');

const multer \= require('multer');

const fetch \= require('node-fetch');

const fs \= require('fs');

const path \= require('path');

const {SpeechClient} \= require('@google-cloud/speech'); // Google Cloud Speech-to-Text

require('dotenv').config();

const app \= express();

const upload \= multer({ dest: 'uploads/' });

// 簡易的なパス

app.post('/process', upload.single('audioFile'), async (req, res) \=\> {

  try {

    const lectureName \= req.body.lectureName || '';

    const file \= req.file;

    if (\!file) {

      return res.status(400).json({ message: '音声ファイルを選択してください。' });

    }

    // 1\) 音声ファイルを文字起こし（Speech-to-Text の例）

    const transcription \= await transcribeAudio(file.path);

    // 2\) Gemini API で要約

    const summary \= await summarizeWithGemini(transcription);

    // 要点（仮の抽出。実際には要約から抽出するか、別APIで抽出）

    const keyPoints \= extractKeyPoints(summary);

    // 一時ファイル削除

    fs.unlinkSync(file.path);

    res.json({

      transcription,

      summary,

      keyPoints

    });

  } catch (err) {

    console.error(err);

    res.status(500).json({ message: '内部エラーが発生しました。' });

  }

});

// 文字起こし関数（Google Cloud Speech-to-Text の例）

async function transcribeAudio(filePath) {

  const client \= new SpeechClient();

  const audioBytes \= fs.readFileSync(filePath).toString('base64');

  const request \= {

    audio: { content: audioBytes },

    config: {

      encoding: 'LINEAR16', // 録音フォーマットに合わせる

      languageCode: 'ja-JP'

    }

  };

  const \[response\] \= await client.recognize(request);

  const transcription \= (response.results || \[\])

    .map(res \=\> res.alternatives\[0\].transcript)

    .join('\\n');

  return transcription;

}

// Gemini API で要約する関数（モデル・エンドポイントは公式ドキュメントを参照）

async function summarizeWithGemini(text) {

  const apiKey \= process.env.GEMINI\_API\_KEY;

  const model \= process.env.GEMINI\_MODEL || 'models/your-gemini-model';

  if (\!apiKey) throw new Error('Gemini API Key not configured.');

  const prompt \= \`以下の講義の文字起こしを要約してください：\\n\\n${text}\`;

  // 実際のエンドポイントは公式ドキュメントを参照してください

  const endpoint \= \`https://gemini.googleapis.com/v1/models/${encodeURIComponent(model)}:predict\`;

  const body \= {

    prompt: { text: prompt }, // 仕様はモデルによって異なる

    max\_tokens: 200, // 例

  };

  const resp \= await fetch(endpoint, {

    method: 'POST',

    headers: {

      'Authorization': \`Bearer ${apiKey}\`,

      'Content-Type': 'application/json'

    },

    body: JSON.stringify(body)

  });

  if (\!resp.ok) {

    const err \= await resp.text();

    throw new Error(\`Gemini API error: ${err}\`);

  }

  const data \= await resp.json();

  // 返却形式はモデルにより異なる。以下は例です

  const text \= data?.choices?.\[0\]?.text || data?.result?.summary || '';

  return text.trim();

}

// 要点抽出（要約から3〜5個の箇条書き風に分割する簡易処理）

function extractKeyPoints(summary) {

  // 簡易実装: 各文を改行または句点で分割して3〜5個を抽出

  if (\!summary) return \[\];

  const sentences \= summary.split(/(?\<=\[。．！\!？\])\\s+/);

  const points \= sentences.slice(0, 5).map(s \=\> s.trim()).filter(Boolean);

  return points;

}

const PORT \= process.env.PORT || 3000;

app.listen(PORT, () \=\> {

  console.log(\`Server running on port ${PORT}\`);

});

