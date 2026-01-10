name: Build Android Manual
on:
  workflow_dispatch:

jobs:
  build-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: master

      - uses: actions/setup-java@v4
        with:
          distribution: "zulu"
          java-version: "17"
          cache: "gradle"

      - name: Flutter action
        uses: subosito/flutter-action@v2
        with:
          channel: 'stable'
          cache: true

      # === 关键修改点开始 ===
      # 使用绝对路径解决 "Not found" 问题
      - name: Generate Dummy Keystore
        run: |
          # 1. 进入 app 目录生成 keystore
          cd simple_live_app/android/app
          keytool -genkey -v -keystore keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias key -storepass 123456 -keypass 123456 -dname "CN=Test, OU=Test, O=Test, L=Test, S=Test, C=US"
          
          # 2. 获取 keystore 的绝对路径
          KEYSTORE_PATH=$(pwd)/keystore.jks
          echo "生成的签名文件路径: $KEYSTORE_PATH"
          
          # 3. 回到 android 目录创建 key.properties
          cd ..
          echo "storePassword=123456" > key.properties
          echo "keyPassword=123456" >> key.properties
          echo "keyAlias=key" >> key.properties
          # 重点：写入绝对路径
          echo "storeFile=$KEYSTORE_PATH" >> key.properties
          
          # 打印检查一下
          cat key.properties
      # === 关键修改点结束 ===

      - name: Restore packages
        run: |
          cd simple_live_app
          flutter pub get

      - name: Build APK
        run: |
          cd simple_live_app
          flutter build apk --release --split-per-abi

      - name: Upload APK to Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: app-release
          path: |
            simple_live_app/build/app/outputs/flutter-apk/*.apk
