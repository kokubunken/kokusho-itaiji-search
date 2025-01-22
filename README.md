# 国書データベースの異体字対応事例紹介
国文研でPGroongaを使い国書データベースの異体字検索を実現した方法を紹介します。

## PGroongaのインストール
[公式ドキュメント](https://pgroonga.github.io/ja/install/)に各プラットフォーム別のインストール方法が記載されておりますので、お手持ちの環境に応じてインストールをお願いします。

### PGroongaのインデックス指定方法について
こちらの同義語検索ではPGroongaのNormalizerTableを使用して実現されています。<br>
詳しくは公式ドキュメントの[NormalizerTableの使い方](https://pgroonga.github.io/ja/reference/create-index-using-pgroonga.html#normalizer-table)を参照願います。

## サンプルデータベース

具体的な異体字対応のデータベースの作り方と検索例を紹介します：

```sql
-- PostgreSQLに必要なPGroonga拡張機能を追加します
CREATE EXTENSION IF NOT EXISTS pgroonga;

-- 異体字対応をさせる辞書テーブルをsynonymsという名前で作ります
CREATE TABLE synonyms (
  target text,
  normalized text
);

-- 異体字対応用のインデックスを作ります
CREATE INDEX pgroonga_synonyms_index ON synonyms
 USING pgroonga (target pgroonga_text_term_search_ops_v2)
                INCLUDE (normalized);

INSERT INTO synonyms VALUES ('笔', '筆');
INSERT INTO synonyms VALUES ('衜', '道');
INSERT INTO synonyms VALUES ('衟', '道');
INSERT INTO synonyms VALUES ('噵', '道');


-- 検索対象となる書誌テーブルをbibliosという名前で作成します
CREATE TABLE biblios (
  title text
);

-- 異体字検索に必要となるPGroongaのインデックスをbibliosテーブルのtitleカラムに作成します
CREATE INDEX pgroonga_titles_index
          ON biblios
       USING pgroonga (title pgroonga_text_full_text_search_ops_v2)
   WITH (normalizers='
                NormalizerNFKC150,
                NormalizerTable(
                  "normalized", "${table:public.pgroonga_synonyms_index}.normalized",
                  "target", "target"
                )
             ');

-- サンプルデータを登録します
INSERT INTO biblios
  (title)
  VALUES ('筆道指南'),('笔道傳授'),('石慴並笔衜書'),('筆衟秘伝'),('笔噵（御手本）'),('并笔道有本篇注解'),('花月堂文并筆道有本篇註解');


-- 「筆道」で検索、どの筆道もヒットする
SELECT title FROM biblios WHERE title &@~ '筆道';
--           title           
-- --------------------------
--  筆道指南
--  笔道傳授
--  石慴並笔衜書
--  筆衟秘伝
--  笔噵（御手本）
--  并笔道有本篇注解
--  花月堂文并筆道有本篇註解
-- (7 rows)


-- 次に「笔衟」で検索している例、同じく全ての筆道がヒットします
SELECT title FROM biblios WHERE title &@~ '笔衟';
--           title           
-- --------------------------
--  筆道指南
--  笔道傳授
--  石慴並笔衜書
--  筆衟秘伝
--  笔噵（御手本）
--  并笔道有本篇注解
--  花月堂文并筆道有本篇註解
-- (7 rows)
```

## 国書データベース異体字SQL

国書データベースで実際に使用している異体字リストのSQLは[kokusho_itaiji.sql](/kokusho_itaiji.sql)で公開しています。

## 応用編

複数文字にも異体字割り当てが出来ますので、例えば下記のような対応も可能です。

```sql
INSERT INTO synonyms VALUES ('かゝみ', 'かかみ');
INSERT INTO synonyms VALUES ('つゝら', 'つつら');
INSERT INTO synonyms VALUES ('なゝくさ', 'なつくさ');
```

検索キーワードが`かゝみ`でも`かかみ`でも双方向に検索可能になります。`つゝら`や`なゝくさ`も同様です。
