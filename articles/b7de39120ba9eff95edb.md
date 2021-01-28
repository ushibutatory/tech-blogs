---
title: "[Unity]WebGL: Server configuration for Nginx"
emoji: "🎮"
type: "tech"
topics: ["nginx","WebGL","Unity"]
published: true
---
Unityの公式には、WebGLのサーバ設定例はIISとApacheしか書いていません。
[WebGL: Server configuration code samples](https://docs.unity3d.com/2020.2/Documentation/Manual/webgl-server-configuration-code-samples.html)

nginx向けに書いて動作確認したものを掲載しておきます。

## 環境など

- Unity 2020.2.0f
- gzip圧縮のみ対応。Brotli圧縮の場合でも同じように書けばよいはず。

:::message
Herokuにホスティングして動作確認していたので、都合上、ポート番号が環境変数になっています。
:::

## nginx.conf

```nginx
server {
  listen $PORT;

  location / {
    root   /webgl; # お好みで
    index  index.html;

    gzip off;

    location ~* \.gz$ {
      add_header Content-Encoding gzip;

      location ~* \.data\.gz$ {
          types { }
          default_type application/octet-stream;
      }
      location ~* \.wasm\.gz$ {
          types { }
          default_type application/wasm;
      }
      location ~* \.js\.gz$ {
          types { }
          default_type application/javascript;
      }
      location ~* \.symbols\.json\.gz$ {
          types { }
          default_type application/octet-stream;
      }
    }
  }
}
```

### content-typeの設定について

若干ハック感ありますが、一応nginxの公式の記法に倣っています。
[Module ngx_http_core_module#types](http://nginx.org/en/docs/http/ngx_http_core_module.html#types)

追加モジュールが必要ですが、以下の方法で設定すると可読性は高くなると思います。お好みで。
[nginxでレスポンスヘッダを書き換える - Qiita](https://qiita.com/reiki4040/items/218438c6e32ba585fd99)

## 余談: Herokuデプロイ用のDockerfile

```dockerfile
FROM nginx:alpine

# copy binary files
WORKDIR /webgl
COPY ./bin .

# configure and run in foreground
WORKDIR /etc/nginx/conf.d
RUN rm default.conf
COPY ./environments/webgl.conf.template .
CMD /bin/ash -c "envsubst '\$PORT' < webgl.conf.template > default.conf" && /usr/sbin/nginx -g "daemon off;"
```

## 全体を通して

もっとこうしたほうがいいぞ、等あればご指摘ください。
