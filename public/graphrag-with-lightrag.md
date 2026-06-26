---
title: GraphRAG を LightRAG で試してみた
tags:
  - RAG
  - GraphRAG
  - LightRAG
  - ナレッジグラフ
  - LLM
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

<!-- メモ: シリーズの発展編として GraphRAG を試す動機。前回「RAG の基礎技術を実装して整理する」のおわりにで予告した発展編のひとつ。LightRAG で自分の Qiita 記事をコーパスにナレッジグラフを構築し、検索から可視化まで試した。rag-sandbox へのリンク。問題→解決の煽りは避けフラットに。 -->

## ナレッジグラフとは

<!-- メモ: ナレッジグラフは RAG 固有の技術ではないという入り。エンティティ(ノード)と関係(エッジ)でグラフ表現するデータベース。用途の広がり(Google 知識パネル / Wikidata / 医療・創薬 / 推薦システム / サイバーセキュリティ)。構築・表現手法の表(RDF/OWL・Property Graph・OpenKE/KGE・LLM 自動抽出)。最後の LLM 自動抽出が今回のアプローチ。 -->

## GraphRAG とは

<!-- メモ: 通常 RAG(チャンクをベクトル化して類似検索)との対比。文書をまたいだ関係や概念のつながりは見えない。GraphRAG は挿入時に LLM でエンティティ・関係を抽出してグラフを作り検索時に辿る。概念的・横断的な問いに強い。Microsoft GraphRAG(2024)はコミュニティ検出・サマリー生成など前処理が重い → 同コンセプトを軽量実装した LightRAG を選んだ。 -->

## LightRAG とは

<!-- メモ: 概要(2024/10 香港大、EMNLP 2025 採択)。挿入フェーズ(チャンク分割→エンティティ・関係抽出→Gleaning 二次抽出→グラフ + ベクトル DB)。クエリフェーズの 5 モード表(naive/local/global/hybrid/mix と向いている問い)。local/global がグラフ検索ならでは。永続化(GraphML + nano-vectordb の JSON、追加サービス不要)。 -->

## 実装

<!-- メモ: この親セクション直下は導入のみ。詳細は ### サブセクションへ。 -->

### 環境とモデル

<!-- メモ: devcontainer(CUDA 12.6 + Python 3.12)/ uv。LLM=Ollama qwen2.5:7b、埋め込み=mxbai-embed-large(1024 次元)、グラフストレージ=NetworkXStorage(GraphML 永続化)。埋め込み次元の注意: ollama_embed はデフォルト 1024 次元前提で nomic-embed-text(768)だと次元不一致でcrash、mxbai-embed-large は 1024 でそのまま使えた。依存追加 uv add lightrag-hku networkx pyvis。 -->

### モジュール（`lightrag_store.py`）

<!-- メモ: src/rag/kg/lightrag_store.py のコード(build_lightrag / insert_documents / query)。llm_model_max_async=1(Ollama はシリアル処理、並列投入でタイムアウト頻発)。逐次 ainsert()(1 件ずつで処理単位を明確に、失敗ドキュメントの再試行挙動)。 -->

### 実験スクリプト

<!-- メモ: experiments/run_lightrag.py。5 モードを順に実行、--skip-insert / --limit / --model / --mode 引数。 -->

### 挿入フェーズで分かったこと

<!-- メモ: 実運用のリアル。エンティティ抽出は重い(1 チャンクに LLM 2 回=一次抽出 + Gleaning、qwen2.5:7b + Ollama で 1 チャンク 30〜120 秒)→ 初回だけ挿入し以降 --skip-insert で再利用。Gleaning タイムアウトと再試行キューの雪だるま。テストは --limit 2〜3 で小規模に。 -->

## 結果

<!-- メモ: この親セクション直下は前提のみ(Next.js App Router + microCMS の記事 2 件を挿入し "Next.js の App Router で microCMS を使う方法は？" をクエリ)。詳細は ### サブセクションへ。 -->

### 5 モードの比較

<!-- メモ: naive(ベクトル検索のみ、チャンクの写経に近い)/ local(エンティティ近傍、ただし中国語応答が出るケース)/ global(コミュニティサマリーで俯瞰)/ hybrid(local+global、最もバランス)/ mix(hybrid+naive、最広コンテキスト)。各モードの回答の違いを簡潔に。コードブロックの引用は要点のみ。 -->

### グラフの可視化

<!-- メモ: 【画像1】全体グラフ(512 ノード・433 エッジ、黄=ハブノード)= _work/lightrag_graph.png を Qiita にアップロードして URL を貼る。2 記事でこの規模。【画像2】クエリサブグラフ(赤=クエリ関連エンティティ、青=近傍)= _work/subgraph_Next.jsのApp_RouterでmicroCMSを使う.png をアップロード。コサイン類似度の上位エンティティ表(microCMS が 0.623 でヒット)。サブグラフ抽出の自前コード(embed_query / load_vdb / cosine_search / extract_subgraph)。 -->

## 気づいたこと・使い分け

<!-- メモ: よかった点(実装が手軽・5 モードの使い分け・グラフの永続化・コーパス内コンテンツへの精度)。課題(挿入コスト・エンティティが英語化・モデル依存・local モードの言語バグ)。通常 RAG との使い分け(チャンクの正確な取り出し vs 概念間の関係を辿る、mix で両取り)。 -->

## おわりに

<!-- メモ: ナレッジグラフが RAG 固有でないという学び。local/global の視点の面白さ、文書の境界を超えて辿る体験。次の実験への展望(コーパス増・日本語エンティティ抽出のプロンプトチューニング)。 -->

## 付録

<!-- メモ: 使用したモデルとバージョンの表(LightRAG 1.5.4 / qwen2.5:7b / mxbai-embed-large 1024 / NetworkXStorage / Python 3.12 / devcontainer CUDA 12.6)。 -->

