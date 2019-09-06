# 调试

![1558798003591](C:\Users\zhanglin\AppData\Roaming\Typora\typora-user-images\1558798003591.png)



# 打包Android APK

* 1、之前的步骤按照官网的教程走  

  [https://reactnative.cn/docs/signed-apk-android/]: https://reactnative.cn/docs/signed-apk-android/

* **2、建立文件夹：`android/app/src/main/assets**`

* **3、运行命令：  `react-native bundle --platform android --dev false --entry-file index.js --bundle-output android/app/src/main/assets/index.android.bundle --assets-dest android/app/src/main/res/**`

* 4、进入android目录，执行命令：`gradlew assembleRelease`



# 安装`react-native-vector-icons`图标

* 执行命令：`npm add react-native-vector-icons`

* 因为该库和android、ios原生库有关联，所以需要执行命令将其与react-native关联起来

  `react-native link react-native-vector-icons`



# REACT-REDUX

![1559403476646](C:\Users\zhanglin\AppData\Roaming\Typora\typora-user-images\1559403476646.png)

![1559403529858](C:\Users\zhanglin\AppData\Roaming\Typora\typora-user-images\1559403529858.png)

![1559403611146](C:\Users\zhanglin\AppData\Roaming\Typora\typora-user-images\1559403611146.png)

![1559403641374](C:\Users\zhanglin\AppData\Roaming\Typora\typora-user-images\1559403641374.png)

![1559403841854](C:\Users\zhanglin\AppData\Roaming\Typora\typora-user-images\1559403841854.png)

