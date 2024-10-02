---
title: "Project | trackVStrack | 크롤링 로직 리팩토링:from JS to TS"
categories: [Projects,tackVStrack]
tags: [ts,problem improvement]
image: ts-logo-256.png
mermaid: true
---

데이터 수집과정이 다소 길고 복잡한 `task`이지만 자바스크립트와 문서정리로 어느정도 원하는바까지는 만들었다. 

하지만  오랜만에 코드를 복기해보며, 또 약간의 수정을 반영할때 생기는 디버깅이슈들을 마주하며 자바스크립트의 한계를 느꼈다. 또 현재는 주간차트만 고려하고있지만, 플랫폼과 차트 스코프를 더 확장해 나갈것을 고려하고 있기에 타입스크립트로 마이그레이트 하였다. 과정에서 느낀바와 개선된점에 대해 글을 남긴다.

이전 구현세부사항은 아래를 참고

- [js version crawling Feature](https://hypnotic-geography-1af.notion.site/Crawling-and-Integrate-Domestic-platforms-41842d71244746b0ba24c0162c1f7e31?pvs=4) 
- [js version crawling readme.md](https://github.com/woong8249/trackVStrack/blob/main/crawling/readme.md)
- [ts version crawling readme.md](https://github.com/woong8249/trackVStrack/blob/main/ts-crawling/readme.md)
    
  
## 개선사항
---

### 1. **빠짐없는 데이터수집**

아래와 같은 기능과 함께 `typescript`의 strict한 rule을 적용하며 리펙토링을 하다보니, 에러포인트를 통제할 수 있었고 어지간하면 데이터가 누락되는경우가 없었다.

- **참여 아티스트가 복수인경우에 대한 크롤링 로직 개선**
- **크롤링 결과 유효성검사 도입**
- **필수데이터 빠진경우 log파일로 쉽게 디버깅 `using winston`**
    - inst이거나, 19금 곡인경우
    ```json
    {"level":"error","lyrics":"missing","message":"Missing required track information in genie","timestamp":"2024-09-27 08:06:47","url":"https://www.genie.co.kr/detail/albumInfo?axnm=80543617"}
    ```
    
- **19금 곡에대한 경우 추가 크롤링 로직 추가**

### 2. **crawling fail시 재시도와 캐싱 적용**

- `axios-retey`
- `cache using map`
- sourcecode
    
    ```tsx
    import axios, { type AxiosRequestConfig } from 'axios';
    import axiosRetry from 'axios-retry';
    import winLogger from '../logger/winston';
    import * as cheerio from 'cheerio';
    
    // 2분
    const MAX_DELAY = 120_000;
    const customRetryDelay = (retryNumber: number) => {
      const delay = axiosRetry.exponentialDelay(retryNumber);
      return Math.min(delay, MAX_DELAY);
    };
    
    axiosRetry(axios, {
      retries: 30,
      retryDelay: customRetryDelay,
      onRetry: (retryCount, error, requestConfig) => {
        winLogger.warn(`Retry attempt ${retryCount.toString()} for ${String(requestConfig.url)}`, { error: error.message });
      },
    });
    
    const delay = (ms: number) => new Promise((resolve) => { setTimeout(resolve, ms); });
    
    const cache = new Map<string, string>();
    export async function getHtml(
      url: string,
      opt: AxiosRequestConfig = { timeout: 300_000 },
      retries: number = 20, // Number of manual retries
      delayFactor: number = 3000, // default delay time
    ) {
      if (cache.has(url)) {
        return cache.get(url) as string; // 캐시된 결과 반환
      }
      try {
        const response = await axios.get(url, opt);
        const $ = cheerio.load(response.data as string);
        const h2Text = $('h2').text();
    
        if (h2Text.includes('blocked')) {
          if (retries > 0) {
            const retryDelay = customRetryDelay(20 - retries);
            winLogger.warn(`Blocked response detected in genie. Retrying in ${retryDelay.toString()} ms... (Remaining retries: ${(retries - 1).toString()})`);
            await delay(retryDelay);
            return await getHtml(url, opt, retries - 1, delayFactor);
          }
          throw new Error('Max retries reached: Blocked response not resolved.');
        }
        cache.set(url, response.data as string);
        return response.data as string;
      } catch (err: unknown) {
        let errorMessage = 'Unknown error';
        if (err instanceof Error) {
          errorMessage = err.message;
        }
    
        winLogger.error('Fetch Fail', { url, opt, error: errorMessage });
        throw new Error(errorMessage);
      }
    }
    
    ```
    

### **3.이미지를 이용한 통합로직 추가**

[이미지 비교방식을 이용한  로직 추가](https://woong8249.github.io/posts/Project-trackVStrack-%EC%9D%B4%EB%AF%B8%EC%A7%80_%EB%B9%84%EA%B5%90%EB%B0%A9%EC%8B%9D%EC%9D%84_%EC%9D%B4%EC%9A%A9%ED%95%9C-_%EB%A1%9C%EC%A7%81-%EC%B6%94%EA%B0%80/) 

### **4.수집된 데이터에 대한 테스트코드 추가**

- Classification test between tracks with the same trackKeyword
- Classification test between artist with the same artistKeyword.

## 결과
---

### **1. 수집연도 확장**
    
    | `crawling.js` | `crawling.ts` |
    | --- | --- |
    | 2019 ⇒ now | 2013 ⇒ now |




### **2. 크롤링 & 검수 과정 축소 , 일관된 스크립트 구성**

- **before**
    1. **Collecting and updating data**
    ```mermaid
    sequenceDiagram
        participant User
        participant Script
    
        User->>Script: pnpm run fetch YYYY-MM-DD YYYY-MM-DD
        Script-->>User: Data collected
        User->>User: Manual inspection (Track lyrics & Artist info)
        
        User->>Script: pnpm run integrate YYYY-MM-DD YYYY-MM-DD
        Script-->>User: Tracks and artists integrated
        User->>User: Manual inspection (trackKey & subFix validation, track grouping)
        
        User->>Script: pnpm run upsert YYYY-MM-DD YYYY-MM-DD
        Script-->>User: DB updated
    ```
    2. **DB migration**
    
    `pnpm run migrate`
    
    수집된 데이터를 한번에 데이터베이스로 이전하거나 데이터 수집 후 데이터베이스 스키마를 조정해야 할 경우 다음 명령어를 사용합니다:
    
- **after**
    1. **Collecting and updating data(specific period)**
        매주 업데이트 되는  차트 크롤링 및 update를 위해 사용
    ```mermaid
    sequenceDiagram
        participant User
        participant APP
    
        User->>APP: pnpm run fetch YYYY-MM-DD YYYY-MM-DD
        APP-->>User: Data collected
        User->>User: inspection with log(If there is missing required information ) 
        User->>APP: pnpm run insert YYYY-MM-DD YYYY-MM-DD
        APP->>User: DB updated
    ```

    2. insertAll

    수집한 모든데이터 한번에 DB insert => `pnpm run insertAll`

    
        
### **3. 더 정확한 분류 검증**
```bash
  Classification test between tracks with the same trackKeyword.
    Even if they have the same trackKeyword,
    they will be classified as different tracks if the lyric similarity, artistName similarity, or trackImage similarity is low. (77) 900ms
     ✓ Check trackKeyword : 'something' , artistNameJoin : '걸스데이 (Girl\'s Day)'
     ✓ Check trackKeyword : 'something' , artistNameJoin : '동방신기 (TVXQ!)'
     ✓ Check trackKeyword : '약속' , artistNameJoin : '이예준'
     ✓ Check trackKeyword : '약속' , artistNameJoin : '김수현'
     ✓ Check trackKeyword : 'hush' , artistNameJoin : 'miss A'
     ✓ Check trackKeyword : 'hush' , artistNameJoin : '브라운아이드걸스'
     ✓ Check trackKeyword : 'lonely' , artistNameJoin : '효린'
     ✓ Check trackKeyword : 'lonely' , artistNameJoin : 'B1A4'
     ✓ Check trackKeyword : 'i love you' , artistNameJoin : '알리 (ALi)임재범'
     ✓ Check trackKeyword : 'i love you' , artistNameJoin : 'Sanha'
     ✓ Check trackKeyword : 'let it go' , artistNameJoin : 'Demi Lovato'
     ✓ Check trackKeyword : 'let it go' , artistNameJoin : '효린'
     ✓ Check trackKeyword : '바람이 분다' , artistNameJoin : '포맨 (4MEN)'
     ✓ Check trackKeyword : '바람이 분다' , artistNameJoin : '김필'
     ✓ Check trackKeyword : '고백' , artistNameJoin : '정준일'
     ✓ Check trackKeyword : '고백' , artistNameJoin : 'Standing Egg (스탠딩 에그)'
     ✓ Check trackKeyword : '고백' , artistNameJoin : '김동률'
     ✓ Check trackKeyword : '안녕' , artistNameJoin : '효린'
     ✓ Check trackKeyword : '안녕' , artistNameJoin : 'AKMU (악뮤)'
     ✓ Check trackKeyword : '안녕' , artistNameJoin : '버즈 (Buzz)'
     ✓ Check trackKeyword : '꽃향기' , artistNameJoin : '임정희'
     ✓ Check trackKeyword : '꽃향기' , artistNameJoin : '최진혁'
     ✓ Check trackKeyword : '숨소리' , artistNameJoin : 'S.M. THE BALLAD'
     ✓ Check trackKeyword : '숨소리' , artistNameJoin : '제국의아이들 (ZE:A)'
     ✓ Check trackKeyword : 'goodbye' , artistNameJoin : '소녀시대 (GIRLS\' GENERATION)'
     ✓ Check trackKeyword : 'goodbye' , artistNameJoin : '임창정'
     ✓ Check trackKeyword : '씨스루' , artistNameJoin : '권진아'
     ✓ Check trackKeyword : '씨스루' , artistNameJoin : '프라이머리'
     ✓ Check trackKeyword : 'home' , artistNameJoin : '버나드 박'
     ✓ Check trackKeyword : 'home' , artistNameJoin : '로이킴'
     ✓ Check trackKeyword : '야생화' , artistNameJoin : '박효신'
     ✓ Check trackKeyword : '야생화' , artistNameJoin : '임도혁 & 장우람'
     ✓ Check trackKeyword : 'your love' , artistNameJoin : '2BIC (투빅)'
     ✓ Check trackKeyword : 'your love' , artistNameJoin : '이정 (J.Lee)'
     ✓ Check trackKeyword : '인연' , artistNameJoin : '윤민수(바이브)신용재 (2F)'
     ✓ Check trackKeyword : '인연' , artistNameJoin : '이선희'
     ✓ Check trackKeyword : '스토커' , artistNameJoin : '매드 클라운 (Mad Clown)'
     ✓ Check trackKeyword : '스토커' , artistNameJoin : '10CM'
     ✓ Check trackKeyword : '십년이 지나도' , artistNameJoin : '권진아'
     ✓ Check trackKeyword : '십년이 지나도' , artistNameJoin : '플라이 투 더 스카이'
     ✓ Check trackKeyword : '내게 남은 세가지' , artistNameJoin : '백아연'
     ✓ Check trackKeyword : '내게 남은 세가지' , artistNameJoin : '강하늘'
     ✓ Check trackKeyword : 'thunder' , artistNameJoin : 'EXO-K'
     ✓ Check trackKeyword : 'thunder' , artistNameJoin : 'EXO-M'
     ✓ Check trackKeyword : 'run' , artistNameJoin : 'EXO-K'
     ✓ Check trackKeyword : 'run' , artistNameJoin : 'EXO-M'
     ✓ Check trackKeyword : 'love, love, love' , artistNameJoin : 'EXO-K'
     ✓ Check trackKeyword : 'love, love, love' , artistNameJoin : 'EXO-M'
     ✓ Check trackKeyword : '오늘따라' , artistNameJoin : '팬텀'
     ✓ Check trackKeyword : '오늘따라' , artistNameJoin : '2am'
     ✓ Check trackKeyword : 'sugar' , artistNameJoin : 'JAMIE (제이미)백예린 (Yerin Baek)'
     ✓ Check trackKeyword : 'sugar' , artistNameJoin : 'Maroon 5'
     ✓ Check trackKeyword : '아름다워' , artistNameJoin : '태양'
     ✓ Check trackKeyword : '아름다워' , artistNameJoin : '이정 (J.Lee)'
     ✓ Check trackKeyword : 'a' , artistNameJoin : 'GOT7 (갓세븐)'
     ✓ Check trackKeyword : 'a' , artistNameJoin : '지누션'
     ✓ Check trackKeyword : '우산' , artistNameJoin : '윤하 (YOUNHA)'
     ✓ Check trackKeyword : '우산' , artistNameJoin : '에픽하이 (EPIK HIGH)'
     ✓ Check trackKeyword : 'i\'m in love' , artistNameJoin : '에일리 (Ailee) & 2LSON'
     ✓ Check trackKeyword : 'i\'m in love' , artistNameJoin : '시크릿 (Secret)'
     ✓ Check trackKeyword : 'darling' , artistNameJoin : '걸스데이 (Girl\'s Day)'
     ✓ Check trackKeyword : 'darling' , artistNameJoin : '에디킴'
     ✓ Check trackKeyword : '전화번호' , artistNameJoin : '스윙스 (Swings)'
     ✓ Check trackKeyword : '전화번호' , artistNameJoin : '지누션'
     ✓ Check trackKeyword : '너를 사랑해' , artistNameJoin : '윤미래'
     ✓ Check trackKeyword : '너를 사랑해' , artistNameJoin : 'S.E.S.'
     ✓ Check trackKeyword : 'lost stars' , artistNameJoin : 'Adam Levine'
     ✓ Check trackKeyword : 'lost stars' , artistNameJoin : 'Maroon 5'
     ✓ Check trackKeyword : 'u' , artistNameJoin : '엠씨더맥스 (M.C the MAX)'
     ✓ Check trackKeyword : 'u' , artistNameJoin : '존박'
     ✓ Check trackKeyword : '우리' , artistNameJoin : '지나'
     ✓ Check trackKeyword : '우리' , artistNameJoin : '토이 (Toy)'
     ✓ Check trackKeyword : '소격동' , artistNameJoin : '아이유 (IU)'
     ✓ Check trackKeyword : '소격동' , artistNameJoin : '서태지'
     ✓ Check trackKeyword : '소격동' , artistNameJoin : '곽진언'
     ✓ Check trackKeyword : '사랑에 빠지고 싶다' , artistNameJoin : '정승환'
     ✓ Check trackKeyword : '사랑에 빠지고 싶다' , artistNameJoin : '김조한'
   ✓
    Classification test between artist with the same artistKeyword.
    Even if they have the same artistKeyword,
    they will be classified as different artist if the artistName similarity, or artistImage similarity is low. (31) 899ms
     ✓ Check artistKeyword : '에일리', artistName :'에일리 (Ailee)'
     ✓ Check artistKeyword : '에일리', artistName :'에일리 (Ailee) & 2LSON'
     ✓ Check artistKeyword : '케이윌', artistName :'케이윌 (K.Will)'
     ✓ Check artistKeyword : '케이윌', artistName :'케이윌'
     ✓ Check artistKeyword : '린', artistName :'린 (LYn)'
     ✓ Check artistKeyword : '린', artistName :'린 (LYn) & 레오 (LEO)'
     ✓ Check artistKeyword : 'exo', artistName :'EXO'
     ✓ Check artistKeyword : 'exo', artistName :'EXO-M'
     ✓ Check artistKeyword : '매드클라운', artistName :'매드 클라운 (Mad Clown)'
     ✓ Check artistKeyword : '매드클라운', artistName :'매드 클라운 (Mad Clown) & 요조'
     ✓ Check artistKeyword : '이정', artistName :'이정'
     ✓ Check artistKeyword : '이정', artistName :'이정 (J.Lee)'
     ✓ Check artistKeyword : '소녀시대', artistName :'소녀시대 (GIRLS\' GENERATION)'
     ✓ Check artistKeyword : '소녀시대', artistName :'소녀시대-태티서 (Girls\' Generation-TTS)'
     ✓ Check artistKeyword : '스윙스', artistName :'스윙스 (Swings)'
     ✓ Check artistKeyword : '스윙스', artistName :'스윙스 (Swings) & 에일리 (AILEE)'
     ✓ Check artistKeyword : '스윙스', artistName :'스윙스 (Swings) & 유성은'
     ✓ Check artistKeyword : 'superjunior', artistName :'SUPER JUNIOR-M (슈퍼주니어-M)'
     ✓ Check artistKeyword : 'superjunior', artistName :'SUPER JUNIOR (슈퍼주니어)'
     ✓ Check artistKeyword : 'nsyoon', artistName :'NS Yoon-G'
     ✓ Check artistKeyword : 'nsyoon', artistName :'NS Yoon-G'
     ✓ Check artistKeyword : '손승연', artistName :'손승연'
     ✓ Check artistKeyword : '손승연', artistName :'손승연'
     ✓ Check artistKeyword : '백지영', artistName :'백지영'
     ✓ Check artistKeyword : '백지영', artistName :'백지영'
     ✓ Check artistKeyword : '다이나믹듀오', artistName :'다이나믹 듀오'
     ✓ Check artistKeyword : '다이나믹듀오', artistName :'다이나믹 듀오'
     ✓ Check artistKeyword : '버벌진트', artistName :'버벌진트 (Verbal Jint)'
     ✓ Check artistKeyword : '버벌진트', artistName :'버벌진트'
     ✓ Check artistKeyword : '포스트맨', artistName :'포스트맨 (POSTMEN)'
     ✓ Check artistKeyword : '포스트맨', artistName :'포스트맨 (Postmen) & 바닐라 어쿠스틱'

 Test Files  1 passed (1)
      Tests  108 passed (108)
   Start at  03:48:21
   Duration  4.06s (transform 1.61s, setup 0ms, collect 2.07s, tests 1.80s, environment 0ms, prepare 38ms)
```



## 느낀점

누가 만들어준 개발환경 위에서 편하게 개발을 해왔던 터라, 개발환경을 조금 가볍게 생각하고 그저 빠르게 개발하고 싶어서 JS로 첫발을 뗐던 것 같다.

하지만 프로젝트가 진행될수록 비동기 작업의 복잡성이 예상보다 훨씬 컸고, `data의 depth`도 생각보다 깊었으며, 외부 요인으로 인한 `error`도 자주 발생했다. 자만이었다..

TypeScript로 전환하고, 엄격한 규칙을 적용하면서 로직을 다시 정리한 결과, 코드의 안정성과 일관성을 확보할 수 있었다. 특히, 데이터 누락 없이 수집과 통합이 가능해지면서 로직의 개선이 크게 이루어졌고, 이에 굉장히 만족한다. 개발 환경이 프로젝트의 성공에 얼마나 중요한지 다시 한 번 실감했다


