# native_add

dart:ffi를 이용하여 플러터 프로젝트에서 C++ 함수를 호출하기 위한 플러그인입니다.  
* 참고자료: https://flutter.dev/docs/development/platform-integration/c-interop  

## Getting Started

1. 플러그인 생성  
```terminal
$ flutter create --platforms=android,ios --template=plugin native_add
$ cd native_add
```

2. C++ 파일을 `ios/Classes/` 안에 추가  
* `ios/` 폴더 안에 작성하는 이유: CocoaPods는 podspec 파일 상위에 위치한 소스를 포함할 수 없지만, Gradle은 ios 폴더를 가리키게 할 수 있습니다.
* FFI 라이브러리는 C symbol만 binding할 수 있기 때문에 C++에서는 `extern C` 표기를 해야 합니다.

3. iOS 설정: Xcode에서 C++ 파일 링크하기  
```terminal
$ cd example/
$ open ios/Runner.xcworkspace
```
`Runner` 폴더 마우스 우클릭 > `Add Files to "Runner"...` > C++ 파일 추가

4. Android 설정(1): `android/CMakeLists.txt` 파일 생성  
```
cmake_minimum_required(VERSION 3.4.1)  # for example

add_library( native_add # 라이브러리 명

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             ../ios/Classes/native_add.cpp )
```

5. Andorid 설정(2): `andriod/build.gradle` 파일에 `externalNativeBuild` section 추가  
```
android {
  // ...
  externalNativeBuild {
    // Encapsulates your CMake build configurations.
    cmake {
      // Provides a relative path to your CMake build script.
      path "CMakeLists.txt"
    }
  }
  // ...
}
```

6. FFI library를 이용해 C++ 코드 로딩하기: native code를 다루기 위한 `DynamicLibrary` 생성  
`lib/native_add.dart`에 아래 코드를 추가합니다.  
```dart
import 'dart:ffi'; // For FFI
import 'dart:io'; // For Platform.isX

final DynamicLibrary nativeAddLib = Platform.isAndroid
    ? DynamicLibrary.open("libnative_add.so")
    : DynamicLibrary.process();
```
Android와 iOS 차이가 있어 플랫폼을 분리해서 생성해야 합니다. (Android는 앞서 `CMakeLists.txt`에서 지정한 `native_add`라는 이름 앞에 접두어 `lib`가 붙은 `.so` 파일을 참조)  

```dart
final int Function(int x, int y) nativeAdd =
  nativeAddLib
    .lookup<NativeFunction<Int32 Function(Int32, Int32)>>("native_add")
    .asFunction();
```
위에서 생성한 `DynamicLibrary`인 `nativeAddLib`에서 `nativeAdd`라는 변수에 `native_add`라는 이름을 가진 `Int32` 타입의 변수 두 개를 받고 반환형은 `Int32`인 값을 함수로 할당하겠다는 의미입니다.

7. 함수 호출(예시 파일: `example/lib/main.dart`)  
```dart
// Inside of _MyAppState.build:
  body: Center(
    child: Text('1 + 2 == ${nativeAdd(1, 2)}'),
  ),
```

## Using plugin in a Flutter project
플러그인을 publish하지 않은 상태에서 Flutter project에 사용하기 위한 방법입니다.
1. 플러그인 폴더를 복사해 Flutter project에 붙여넣기  
```
native_add/
sampleapp/
├───android/
├───ios/
├───lib/
├───...
├───pubspec.yaml
```

2. `pubspec.yaml`에 플러그인 path 추가
```yaml
dependencies:
  flutter:
    sdk: flutter
  native_add:
    path: ../native_add
```

3. 함수 호출
