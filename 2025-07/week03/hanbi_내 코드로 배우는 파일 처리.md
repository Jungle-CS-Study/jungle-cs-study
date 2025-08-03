# 내 코드로 배우는 파일 처리

# 🚨 문제 1: 이미지 경로 404 오류

## 초기 코드와 문제점

```javascript
const declarationData = {
  image: {
    imageSrc: './src/assets/brave.jpg',  // ❌ 404 Error
    imageAlt: '용기있게 나오세요',
  },
};
```

브라우저 Network 탭을 확인하면 `GET http://localhost:3000/src/assets/brave.jpg` 요청이 `404 Not Found`를 반환한다. 브라우저가 Document Root(`public/`) 외부의 `src/` 폴더에 접근할 수 없기 때문이다. 브라우저의 Same-Origin Policy와 Document Root 제한으로 인해 `src/` 폴더는 브라우저가 직접 접근할 수 없는 영역인 것이다. 

## 해결책 1: Public 디렉토리 활용

```javascript
// 파일 이동: src/assets/brave.jpg → public/images/brave.jpg
const declarationData = {
  image: {
    imageSrc: '/images/brave.jpg',  // ✅ Document Root 기준 절대 경로
    imageAlt: '용기있게 나오세요',
  },
};
```

## 해결책 2: ES6 모듈 Import (권장)

```javascript
import braveImage from '../assets/brave.jpg';  // Vite가 빌드 시 최적화 처리

const declarationData = {
  image: {
    imageSrc: braveImage,  // ✅ 빌드된 최적화 경로 사용
    imageAlt: '용기있게 나오세요',
  },
};
```

# 🚨 문제 2: 정적 이미지의 동적 변경 한계

## 문제 상황

```javascript
const Declaration = () => {
  const declarationData = {
    image: {
      imageSrc: '/images/brave.jpg',  // 고정된 경로
      imageAlt: '용기있게 나오세요',
    },
  };

  // 사용자가 이미지를 바꾸려면? 방법이 없음
  return (
    <section className="program-section" id="session3">
      <img
        src={declarationData.image.imageSrc}
        alt={declarationData.image.imageAlt}
        className="concept-image"
      />
    </section>
  );
};
```

React에서 UI 변경을 위해서는 상태(State) 변경이 필요하다. 일반 변수를 변경해서는 리렌더링이 트리거 되지 않는다. 

## 해결: React State를 통한 동적 관리

```javascript
const Declaration = () => {
  // React 상태로 이미지 경로 관리
  const [currentImage, setCurrentImage] = useState('/images/brave.jpg');

  const declarationData = {
    sectionInfo: {
      title: '3. 선언문 발표',
      subtitle: '용기 있게 나오세요!',
    },
    image: {
      imageSrc: currentImage,  // ✅ 동적 상태값 사용
      imageAlt: '용기있게 나오세요',
    },
  };

  return (
    <section className="program-section" id="session3">
      <div className="section-header">
        <h2>{declarationData.sectionInfo.title}</h2>
        <p>{declarationData.sectionInfo.subtitle}</p>
      </div>
      
      <img
        src={declarationData.image.imageSrc}
        alt={declarationData.image.imageAlt}
        className="concept-image"
      />
      
      {/* 상태 변경을 통한 이미지 교체 테스트 */}
      <button onClick={() => setCurrentImage('/images/alternative.jpg')}>
        이미지 변경
      </button>
    </section>
  );
};
```

- **동작 원리**: `setCurrentImage` 호출 → React의 Reconciliation 알고리즘 실행 → Virtual DOM 비교 → `<img src>` 속성 변경 감지 → DOM 업데이트 → 화면 리렌더링

# 🚨 문제 3: 사용자 파일 업로드 처리의 복잡성

# 초기 접근의 한계

```javascript
const Declaration = () => {
  const [currentImage, setCurrentImage] = useState('/images/brave.jpg');

  const handleFileChange = (event) => {
    const file = event.target.files[0];
    console.log(file); // File 객체: 메타데이터만 포함, 실제 이미지 데이터 없음
    
    // setCurrentImage(file); // ❌ File 객체는 img src에 사용 불가
    // setCurrentImage(file.name); // ❌ 파일명만으로는 이미지 표시 불가
  };

  return (
    <section>
      <img src={currentImage} className="concept-image" />
      <input type="file" onChange={handleFileChange} />
    </section>
  );
};
```

1. 브라우저 Sandbox로 인해 로컬 파일 경로를 `src`에 직접 사용할 수 없음
2. File 객체는 메타데이터만 포함하므로 실제 파일 내용을 읽는 별도 과정 필요
3. 파일 읽기는 시간이 걸리는 I/O 작업이므로 비동기 처리 필요

## 해결: FileReader API와 Data URL 활용

```javascript
const Declaration = () => {
  const [currentImage, setCurrentImage] = useState('/images/brave.jpg');

  const handleFileSelect = (event) => {
    const file = event.target.files[0];
    if (!file) return;

    // 1. MIME Type 검증
    if (!file.type.startsWith('image/')) {
      alert('이미지 파일만 업로드 가능합니다.');
      return;
    }

    // 2. 파일 크기 검증 (Base64 변환 시 33% 증가 고려)
    const maxSize = 5 * 1024 * 1024; // 5MB
    if (file.size > maxSize) {
      alert('파일 크기는 5MB 이하만 가능합니다.');
      return;
    }

    // 3. FileReader API로 비동기 파일 읽기
    const reader = new FileReader();
    
    // 4. Event-driven Programming: 읽기 완료 시 콜백 실행
    reader.onload = (e) => {
      const dataURL = e.target.result; // Base64 인코딩된 Data URL
      setCurrentImage(dataURL); // React 상태 업데이트 → 리렌더링
    };
    
    reader.onerror = () => {
      alert('파일 읽기 중 오류가 발생했습니다.');
    };
    
    // 5. Non-blocking I/O: 비동기 읽기 시작
    reader.readAsDataURL(file);
  };

  const declarationData = {
    sectionInfo: {
      title: '3. 선언문 발표',
      subtitle: '용기 있게 나오세요!',
    },
    image: {
      imageSrc: currentImage,
      imageAlt: '용기있게 나오세요',
    },
  };

  return (
    <section className="program-section" id="session3">
      <div className="section-header">
        <h2>{declarationData.sectionInfo.title}</h2>
        <p>{declarationData.sectionInfo.subtitle}</p>
      </div>

      <img
        src={declarationData.image.imageSrc}
        alt={declarationData.image.imageAlt}
        className="concept-image"
      />

      <div className="upload-controls">
        <input
          type="file"
          accept="image/*"  // HTML5 MIME Type 필터
          onChange={handleFileSelect}
        />
      </div>
    </section>
  );
};
```

## **생성되는 Data URL 예시**

```other
data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAYEBQYFBAYGBQYHBwYIChAKCgkJ...
```

# 🚨 문제 4: 컴포넌트 책임 분리와 재사용성

## 문제점 진단

현재 Declaration 컴포넌트가 다음과 같은 여러 책임을 동시에 가지고 있었다. 

- UI 렌더링
- 파일 업로드 처리
- 파일 검증 로직
- 에러 처리

Single Responsibility Principle에 따라 각 컴포넌트는 하나의 명확한 책임만 가져야 한다. 

## 해결: Lifting State Up 패턴으로 책임 분리

```javascript
// ImageUpload.js - 파일 업로드 전용 컴포넌트
const ImageUpload = ({ onImageChange, buttonText = '이미지 바꾸기' }) => {
  const handleFileSelect = (event) => {
    const file = event.target.files[0];
    if (!file) return;

    // 파일 검증 로직
    if (!file.type.startsWith('image/')) {
      alert('이미지 파일만 업로드할 수 있어요! 😭');
      return;
    }

    if (file.size > 5 * 1024 * 1024) {
      alert('파일 크기는 5MB 이하만 가능합니다.');
      return;
    }

    // 비동기 파일 읽기
    const reader = new FileReader();
    reader.onload = (e) => {
      onImageChange(e.target.result); // 부모 컴포넌트로 결과 전달
    };
    reader.onerror = () => {
      alert('파일 읽기 중 오류가 발생했습니다.');
    };
    reader.readAsDataURL(file);
  };

  return (
    <div className="upload-controls">
      <label className="upload-button">
        <input
          type="file"
          accept="image/*"
          onChange={handleFileSelect}
          className="upload-input"
        />
        {buttonText}
      </label>
    </div>
  );
};

// Declaration.js - UI 렌더링과 상태 관리만 담당
const Declaration = () => {
  const [currentImage, setCurrentImage] = useState('/images/brave.jpg');

  const declarationData = {
    sectionInfo: {
      title: '3. 선언문 발표',
      subtitle: '용기 있게 나오세요!',
    },
    image: {
      imageSrc: currentImage,
      imageAlt: '용기있게 나오세요',
    },
  };

  return (
    <section className="program-section" id="session3">
      <div className="section-header">
        <h2>{declarationData.sectionInfo.title}</h2>
        <p>{declarationData.sectionInfo.subtitle}</p>
      </div>

      <img
        src={declarationData.image.imageSrc}
        alt={declarationData.image.imageAlt}
        className="concept-image"
      />

      {/* 재사용 가능한 컴포넌트로 분리 */}
      <ImageUpload onImageChange={setCurrentImage} />
    </section>
  );
};
```

## 전체 데이터 플로우 이해하기

```other
1. 사용자 파일 선택 (input[type="file"])
   ↓
2. handleFileSelect 실행 (ImageUpload 컴포넌트)
   ↓ [MIME Type, 파일 크기 검증]
3. FileReader.readAsDataURL() 호출 (Non-blocking I/O)
   ↓ [백그라운드에서 Base64 변환 진행]
4. reader.onload 콜백 실행 (Event-driven)
   ↓ [Data URL 생성 완료]
5. onImageChange(dataURL) 호출 (Lifting State Up)
   ↓ [부모 컴포넌트로 상태 전달]
6. setCurrentImage(dataURL) 실행 (React State)
   ↓ [상태 변경 감지]
7. React Reconciliation 알고리즘 실행
   ↓ [Virtual DOM 비교 및 업데이트]
8. <img src={currentImage}> DOM 업데이트
   ↓
9. 브라우저에 새 이미지 렌더링 완료
```

# 👇 여기서 부터는 안 읽으셔도 됩니다.👇 </br> (부록) 문제를 해결하는데 도움이 됐던 개념들

# 1. 브라우저의 파일 접근 제한

웹 개발을 하다 보면 "분명히 이미지 파일이 있는데 왜 404 에러가 뜨지?"라는 답답한 경험을 하게 된다. 로컬에서는 파일 탐색기로 직접 확인할 수 있는 파일인데, 브라우저에서는 접근할 수 없다니. 이런 현상이 발생하는 이유를 이해하려면 브라우저의 보안 모델을 알아야 한다. 

## **Same-Origin Policy (동일 출처 정책)**

1990년대 초 웹이 발전하면서 악성 웹사이트가 사용자 모르게 다른 사이트의 데이터를 훔치는 공격이 등장했다. 이를 방지하기 위해 브라우저는 "같은 출처의 리소스만 접근 허용"이라는 철칙을 만들었다. 

출처(Origin)는 세 가지 요소로 구성된다. 

- **프로토콜**: `http://` vs `https://`
- **도메인**: `localhost` vs `example.com`
- **포트**: `:3000` vs `:8080`

```javascript
// 현재 페이지: http://localhost:3000
// ✅ 허용: http://localhost:3000/api/data
// ❌ 차단: http://localhost:8080/api/data (다른 포트)
// ❌ 차단: https://localhost:3000/api/data (다른 프로토콜)
```

## **Document Root (문서 루트)**

웹 서버는 보안상 특정 폴더만을 외부에 공개한다. 이 기준점이 바로 Document Root이다. 마치 회사에서 "기밀 문서는 금고에, 공개 자료만 로비에 비치"하는 것과 같은 원리다. 

```other
실제 서버 파일 구조
├── /home/user/secret/         ← 접근 불가 (민감한 설정 파일들)
├── /var/www/html/            ← Document Root (공개 파일들)
│   ├── index.html
│   └── images/
└── /etc/config/              ← 접근 불가 (시스템 설정)
```

개발 환경에서도 동일한 원리가 적용된다. 

```other
React 프로젝트 구조
├── src/                      ← 개발자 작업 공간 (브라우저 접근 불가)
│   ├── components/
│   └── assets/brave.jpg      ← 이 파일을 직접 요청하면 404!
├── public/                   ← Document Root (브라우저 접근 가능)
│   ├── index.html
│   └── images/
└── package.json              ← 프로젝트 설정 (접근 불가)
```

## **Sandbox (샌드박스)**

2000년대 초반, 악성 웹사이트가 사용자 컴퓨터의 파일을 삭제하거나 개인정보를 훔치는 사고가 빈발했다. 이후 브라우저는 웹사이트를 "격리된 감옥"에서 실행하는 샌드박스 모델을 도입했다. 

```javascript
// ❌ 브라우저에서는 불가능한 작업들
const fs = require('fs'); // Node.js 파일 시스템 모듈 사용 불가
fs.readFile('C:/Users/myfile.txt'); // 로컬 파일 직접 읽기 불가
window.location = 'file:///C:/secret.txt'; // 로컬 파일 URL 접근 불가
```

# **2. HTTP 리소스 요청의 작동 원리**

이런 경험도 있다. "같은 파일인데 왜 어떤 경로는 되고 어떤 경로는 안 되지?”. 브라우저가 경로를 해석하는 방식과 웹 서버의 파일 제공 규칙 때문이다. 

## **경로 해석의 실제 과정**

브라우저는 HTML에서 리소스 경로를 만나면 다음과 같은 단계를 거쳐 실제 HTTP 요청을 만든다. 

```javascript
// 현재 페이지: http://localhost:3000/dashboard
// 상대 경로 해석 과정
imageSrc: './assets/hero.jpg'
// ↓ 브라우저 내부 계산
// 현재 경로 + 상대 경로 = http://localhost:3000/dashboard/assets/hero.jpg
// ↓ HTTP 요청 생성  
// GET /dashboard/assets/hero.jpg HTTP/1.1
// Host: localhost:3000
```

문제는 개발자가 의도한 경로와 브라우저가 해석한 경로가 다를 때 발생한다. 

```javascript
// 개발자 의도: src 폴더의 파일을 가져오고 싶음
imageSrc: './src/assets/hero.jpg'

// 브라우저 해석: "현재 위치에서 src 폴더를 찾아보자"
// GET /src/assets/hero.jpg

// 서버 응답: "Document Root(public/)에 src 폴더가 없네요"
// HTTP/1.1 404 Not Found
```

## **절대 경로 vs 상대 경로의 차이**

```javascript
// 절대 경로: Document Root부터 시작
imageSrc: '/images/logo.png'
// → GET /images/logo.png (항상 동일한 요청)

// 상대 경로: 현재 위치에 따라 달라짐
imageSrc: './images/logo.png'
// /home 페이지에서: GET /home/images/logo.png  
// /about 페이지에서: GET /about/images/logo.png
// /products/detail 페이지에서: GET /products/detail/images/logo.png
```

## 실제 개발에서 자주 겪는 문제들:

```javascript
// 문제 상황 1: 페이지별로 이미지가 다르게 보임
const ProfilePage = () => (
  <img src="./avatar.jpg" /> // /profile/avatar.jpg 요청
);

const SettingsPage = () => (
  <img src="./avatar.jpg" /> // /settings/avatar.jpg 요청 (다른 파일!)
);

// 해결책: 절대 경로 사용
const ProfilePage = () => (
  <img src="/images/avatar.jpg" /> // 항상 /images/avatar.jpg 요청
);
```

# **3. React의 상태 관리 메커니즘**

jQuery 시대에는 DOM을 직접 조작했다고 한다. "버튼 클릭하면 이미지 src 속성을 바꾸자"는 식으로 말이다. 하지만 앱이 복잡해지면서 "어디서 뭐가 바뀌었는지 추적하기 어렵다"는 문제가 생겼다. React는 이를 해결하기 위해 상태 중심의 접근법을 도입했다. 

## **State vs Variable**

```javascript
// jQuery 방식 (DOM 직접 조작)
let currentImage = '/images/default.jpg';
$('#myImage').attr('src', currentImage); // 화면 직접 변경

function changeImage() {
  currentImage = '/images/new.jpg'; // 변수만 변경
  $('#myImage').attr('src', currentImage); // 화면도 수동으로 변경해야 함
}
```

```javascript
// React 방식 (상태 기반 선언적 렌더링)
const [currentImage, setCurrentImage] = useState('/images/default.jpg');

// JSX에서 상태 사용
<img src={currentImage} /> // 상태가 바뀌면 자동으로 화면 업데이트

function changeImage() {
  setCurrentImage('/images/new.jpg'); // 상태만 변경하면 끝!
  // React가 자동으로 화면 업데이트
}
```

## 일반 변수로는 왜 화면이 안 바뀔까?

```javascript
const MyComponent = () => {
  let count = 0; // 일반 변수

  const handleClick = () => {
    count++; // 변수는 증가하지만...
    console.log(count); // 콘솔에는 제대로 출력됨
    // 하지만 화면은 여전히 0을 표시 (React가 재렌더링하지 않음)
  };

  return <div>{count}</div>; // 항상 초기값 0만 표시
};
```

## **Reconciliation (재조정)**

DOM 조작은 브라우저에서 가장 비싼 연산 중 하나다. React는 이를 최적화하기 위해 Virtual DOM이라는 중간 단계를 만들었다. 

```javascript
// 상태 변경 전 Virtual DOM
<div>
  <img src="/images/old.jpg" />
  <p>사용자: 김철수</p>
  <span>점수: 100</span>
</div>

// 상태 변경 후 Virtual DOM  
<div>
  <img src="/images/new.jpg" />  ← 변경됨
  <p>사용자: 김철수</p>           ← 동일
  <span>점수: 100</span>          ← 동일
</div>

// React의 Diff 알고리즘 결과
// → img의 src 속성만 실제 DOM에서 업데이트
// → p와 span은 건드리지 않음 (성능 최적화)
```

## Lifting State Up

컴포넌트가 분리되면서 생기는 자연스러운 문제

```javascript
// 문제 상황: 형제 컴포넌트 간 데이터 공유 불가
const ImageDisplay = () => {
  const [image, setImage] = useState('/default.jpg');
  return <img src={image} />;
};

const ImageUpload = () => {
  const handleUpload = (newImage) => {
    // ImageDisplay의 image 상태를 어떻게 바꾸지?
    // 직접 접근할 방법이 없음!
  };
  return <input type="file" onChange={handleUpload} />;
};
```

```javascript
// 해결책: 공통 부모에서 상태 관리
const ImageManager = () => {
  const [image, setImage] = useState('/default.jpg'); // 부모가 상태 소유

  return (
    <div>
      <ImageDisplay image={image} />           {/* props로 상태 전달 */}
      <ImageUpload onImageChange={setImage} /> {/* 콜백으로 상태 변경 권한 전달 */}
    </div>
  );
};
```

4. 비동기 파일 처리

초기 웹은 단순한 문서 공유가 목적이었다. 하지만 파일 업로드, 이미지 처리 등 복잡한 작업이 생기면서 "작업 중에도 페이지가 멈추지 않게 하자"는 요구가 생겼다. 

## **Non-blocking I/O**

```javascript
// Blocking I/O (동기): 옛날 방식
console.log('1. 파일 읽기 시작');
const data = readFileSync('large-file.jpg'); // 여기서 멈춤 (3초 대기)
console.log('2. 파일 읽기 완료');
console.log('3. 다른 작업 시작');
// 결과: 3초 동안 아무것도 할 수 없음

// Non-blocking I/O (비동기): 현재 방식  
console.log('1. 파일 읽기 시작');
readFile('large-file.jpg', (data) => {
  console.log('2. 파일 읽기 완료'); // 3초 후 실행
});
console.log('3. 다른 작업 시작'); // 즉시 실행
// 결과: 파일 읽는 동안에도 다른 작업 가능
```

## **Event-driven Programming**

브라우저 환경에서는 언제 어떤 일이 일어날지 예측하기 어렵다. 사용자가 언제 파일을 선택할지, 네트워크 요청이 언제 완료될지 모르기 때문이다. 

```javascript
// 파일 업로드의 실제 이벤트 흐름
const fileInput = document.querySelector('input[type="file"]');

fileInput.addEventListener('change', (event) => {
  const file = event.target.files[0];
  
  const reader = new FileReader();
  
  reader.addEventListener('loadstart', () => {
    console.log('읽기 시작!');
  });
  
  reader.addEventListener('progress', (e) => {
    console.log(`진행률: ${(e.loaded / e.total) * 100}%`);
  });
  
  reader.addEventListener('load', (e) => {
    console.log('읽기 완료!', e.target.result);
  });
  
  reader.addEventListener('error', () => {
    console.log('읽기 실패!');
  });
  
  reader.readAsDataURL(file);
});
```

# **5. 파일 인코딩과 데이터 표현**

웹의 초기에는 텍스트만 주고받았다. 하지만 이미지, 비디오 등 바이너리 데이터를 전송해야 하는 상황이 생기면서 "어떻게 0과 1을 안전하게 전송할까?"라는 문제가 대두되었다. 

## **MIME Type: 파일 식별의 표준**

파일 확장자만으로는 실제 파일 내용을 보장할 수 없다. 악의적으로 `.jpg` 확장자를 가진 실행 파일을 만들 수도 있으니까. 

```javascript
// 파일 확장자 vs MIME Type
const file = event.target.files[0];

console.log(file.name); // "photo.jpg" (확장자로는 이미지 같음)
console.log(file.type); // "application/x-msdownload" (실제로는 실행 파일!)

// 안전한 검증 방법
if (!file.type.startsWith('image/')) {
  alert('이미지 파일만 업로드 가능합니다!');
  return;
}
```

## **Base64 Encoding**

과거 이메일 시스템은 텍스트만 전송 가능하도록 설계되었다. 하지만 이미지 첨부 요구사항이 생겨났고, 개발자들은 방법을 찾아냈다. Base64가 그 해답이다. 

```javascript
// 원본 바이너리 데이터 (3바이트)
// 01001000 01100101 01101100

// Base64 변환 과정
// 1. 3바이트를 4그룹으로 나눔 (6비트씩)
// 010010 | 000110 | 010101 | 101100

// 2. 각 6비트를 Base64 문자로 변환
// 010010(18) → S
// 000110(6)  → G  
// 010101(21) → V
// 101100(44) → s

// 결과: "SGVs" (4문자)
```

## 메모리 사용량이 증가하는 이유

```javascript
// 1MB 이미지 파일
const originalSize = 1024 * 1024; // 1,048,576 bytes

// Base64 변환 후
const base64Size = Math.ceil(originalSize * 4/3); // 1,398,101 bytes
const increase = base64Size - originalSize; // 349,525 bytes 증가 (33.3%)

console.log(`원본: ${originalSize}bytes, Base64: ${base64Size}bytes`);
console.log(`증가량: ${(increase/originalSize*100).toFixed(1)}%`);
```

## **Data URL: URL에 파일 내용 직접 포함**

일반적인 URL은 "어디에 파일이 있는지"를 알려주는 주소다. Data URL은 "파일 내용이 뭔지"를 직접 포함한다. 

```javascript
// 일반 URL (파일 위치 정보)
<img src="/images/photo.jpg" />
// → 브라우저: "서버야, /images/photo.jpg 파일 보내줘"
// → 서버: "파일 내용을 전송합니다"

// Data URL (파일 내용 직접 포함)  
<img src="data:image/jpeg;base64,/9j/4AAQSkZJRgABA..." />
// → 브라우저: "아, 파일 내용이 여기 다 있네. 바로 표시하자"
// → 서버 요청 없음!
```

Data URL의 장단점

```javascript
// 장점: 서버 요청 없이 즉시 표시
const dataURL = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8/5+hHgAHggJ/PchI7wAAAABJRU5ErkJggg==';
// 1x1 픽셀 투명 이미지 (87바이트) → 즉시 렌더링

// 단점: HTML 크기 증가로 초기 로딩 지연
const largeDataURL = 'data:image/jpeg;base64,' + '매우매우긴Base64문자열...';  
// 1MB 이미지 → HTML이 1.33MB 커짐
```