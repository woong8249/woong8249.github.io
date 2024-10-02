---
title: "Project | trackVStrack | 이미지 비교방식을 이용한 로직 추가, 통합방식 개선"
categories: [Projects,tackVStrack]
tags: [resemble.js,problem improvement]
image: https://raw.github.com/rsmbl/Resemble.js/master/demoassets/resemble.png
---

## 요약

음원 스트리밍 플랫폼(`멜론`,`지니`,`벅스`)간 `track`과 `artist`를 통합시키는 작업을 개선.

플랫폼별 `artistName`, `titleName` 표기법이 조금씩 상이함. 
해서 정규표현식을 거친 `artistKeyword` ,`trackKeyword`를 만들어 비교했음.
동명의 아티스트나 동명의 곡이있기때문에 아래와 같이 추가적인 로직을 거침.

- track통합시`trackKeyword` 가 같은경우 `artistKeyword`와 `lyrics` 의 유사도 ,`debut`를 비교.
    - 이방식은 대부분의 경우 잘 되지만 그럼에도 엣지케이스는 많았음
- artist 통합시 `artistKeyword` 가 같은경우 artistName 의 유사도 비교
    - 아마 분류과정에서 잘못된케이스가 매우 많았을것으로 예상

이미지 비교방식(`resemblejs`)을 조건부로 사용하여 위의 통합과정을 매우개선

| command | before | after |
| --- | --- | --- |
| `SELECT COUNT(*) FROM Artist;` | 2735 | 2375 |
| `SELECT artistKeyword, COUNT(*) AS count FROM Artist GROUP BY artistKeyword HAVING COUNT(*) >= 2;` | 470 | 183 |
| `SELECT COUNT(**) FROM track;*` | 10644 | 10291  |
| `SELECT trackKeyword, COUNT(*) AS count  FROM track  GROUP BY trackKeyword  HAVING COUNT(*) >= 2;` | 1245 | 1028 |

## 기존  통합방식과 한계

---

아래는 아티스트 통합로직의 일부다 . 플랫폼이 다른경우 고유ID를 비교할수없기에 기존에는 artistName 유사도와 , debut를 이용했었다.

```tsx
 	  // 2번 다른 플랫폼인경우
      // artistName 유사도비교
      // debut 비교로직 => conditional
        const similarityList: [Artist, number, boolean | undefined][] = existingArtists.map((existingArtist) => {
        const availablePlatform = existingArtist.platforms.melon || existingArtist.platforms.genie || existingArtist.platforms.bugs as ArtistWithAddInfo;
        const result: [Artist, number, boolean | undefined] = [existingArtist, checkSS(availablePlatform.artistName, artist.artistName), undefined];
        if (availablePlatform.debut !== 'missing' && artist.debut !== 'missing') {
          result[2] = availablePlatform.debut.split('.')[0] === artist.debut.split('.')[0];
        }
        return result;
      });
      const maxArtistSimilarity = similarityList.reduce((pre, cur) => (pre[1] > cur[1] ? pre : cur));
      const condition1 = maxArtistSimilarity[1] >= 0.3;
      const condition2 = maxArtistSimilarity[2] === undefined ? true : maxArtistSimilarity[2];
      if (condition1 && condition2) {
        Object.assign(maxArtistSimilarity[0].platforms, { [platformName]: artist });
        await transactionalEntityManager.save(maxArtistSimilarity[0]);
        continue; //  루프 넘어가기
      }
```

위 로직대로 통합할시 레코드 정보는 아래와같다.

```bash
mysql> SELECT COUNT(*) FROM Artist;
+----------+
| COUNT(*) |
+----------+
|     2735 |
+----------+
1 row in set (0.00 sec)

mysql> SELECT artistKeyword, COUNT(*) AS count FROM Artist GROUP BY artistKeyword HAVING COUNT(*) >= 2;
+-----------------------+-------+
| artistKeyword         | count |
+-----------------------+-------+
| 소녀시대                |     3 |
| 백지영                  |     2 |
| 효린                  |     3 |
| 전효성                |     2 |
| 현아                  |     3 |
| 니콜                  |     3 |
	......
| 범진                  |     2 |
+-----------------------+-------+
470 rows in set (0.01 sec)
```

또한 아래와 같은경우로 인해 엣지케이스가 생김을 확인했다.

1. **사이트별 debut가 다른경우**
    - [효린 debut in genie  =>2011](https://www.genie.co.kr/search/searchMain?query=%25ED%259A%25A8%25EB%25A6%25B0)
    - [효린 debut in bugs => 2010](https://music.bugs.co.kr/artist/80069634?wl_ref=list_ar_01_search)
    - [효린 debut in melon ⇒ 2013.11.26](https://www.melon.com/search/total/index.htm?q=%ED%9A%A8%EB%A6%B0&section=&mwkLogType=T)
    - source
        
        ```json
        [
                {
                    "id": 3,
                    "artistKeyword": "효린",
                    "platforms": {
                        "bugs": {
                            "artistKeyword": "효린",
                            "artistName": "효린",
                            "artistID": "80069634",
                            "artistImage": "https://image.bugsm.co.kr/artist/images/200/800696/80069634.jpg?version=20240810002131.0",
                            "debut": "2010"
                        }
                    }
                },
                {
                    "id": 792,
                    "artistKeyword": "효린",
                    "platforms": {
                        "genie": {
                            "artistKeyword": "효린",
                            "artistName": "효린",
                            "artistID": "80145352",
                            "artistImage": "https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/080/145/352/80145352_1723439966398_23_600x600.JPG",
                            "debut": "2011"
                        }
                    }
                },
                {
                    "id": 1000,
                    "artistKeyword": "효린",
                    "platforms": {
                        "melon": {
                            "artistKeyword": "효린",
                            "artistName": "효린",
                            "artistID": "573812",
                            "artistImage": "https://cdnimg.melon.co.kr/cm2/artistcrop/images/005/73/812/573812_20240812154123_500.jpg?d4ac9f4e2552c75231b1e1c46fa412ec/melon/resize/416/quality/80/optimize",
                            "debut": "2013.11.26"
                        }
                    }
                }
            ]
        ```
        
2. **표기법에 너무 큰차이가 있는경우**
    - [제아 in genie  =>"제아 (브라운아이드걸스)"](https://www.genie.co.kr/search/searchMain?query=%25EC%25A0%259C%25EC%2595%2584#)
    - [제아 in melon => "제아"](https://www.melon.com/artist/timeline.htm?artistId=230519) 
    - source
        
        ```json
        
            [
                {
                    "id": 8,
                    "artistKeyword": "제아",
                    "platforms": {
                        "bugs": {
                            "artistKeyword": "제아",
                            "artistName": "제아",
                            "artistID": "80020174",
                            "artistImage": "https://image.bugsm.co.kr/artist/images/200/800201/80020174.jpg?version=20240620002121.0",
                            "debut": "2006"
                        }
                    }
                },
                {
                    "id": 807,
                    "artistKeyword": "제아",
                    "platforms": {
                        "genie": {
                            "artistKeyword": "제아",
                            "artistName": "제아 (브라운아이드걸스)",
                            "artistID": "53594251",
                            "artistImage": "https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/053/594/251/53594251_1718763462293_14_600x600.JPG",
                            "debut": "2006"
                        }
                    }
                },
                {
                    "id": 999,
                    "artistKeyword": "제아",
                    "platforms": {
                        "melon": {
                            "artistKeyword": "제아",
                            "artistName": "제아",
                            "artistID": "230519",
                            "artistImage": "https://cdnimg.melon.co.kr/cm2/artistcrop/images/002/30/519/230519_20240619110855_500.jpg?06aa1bc1ac27d0b8e73893145842c4af/melon/resize/416/quality/80/optimize",
                            "debut": "2013.01"
                        }
                    }
                }
            ]
        
        ```
        

## **이미지 비교방식**

---

갑자기 희번뜩 이미지를 비교해보면 어떨까라는 생각이들었다. 상식적으로 생각해도 프로필이미지는 싱크시킬 확률이 매우 높을 것이다.

(모든 케이스를 확인하지는 못했지만 대부분의경우 플랫폼별 프로필이미지는 같았다.)

- 빠르게 이미지 비교로직 & 테스트코드작성 ⇒ 예상대로 작동함을 확인
    
    ```tsx
    /* eslint-disable @typescript-eslint/no-non-null-assertion */
    import axios from 'axios';
    import resemble from 'resemblejs';
    import sharp from 'sharp';
    
    async function downloadImageAsBuffer(url: string) {
      const response = await axios<Buffer>({
        url,
        responseType: 'arraybuffer',
      });
      return Buffer.from(response.data);
    }
    
    // 이미지 크기를 동일하게 조정하는 함수
    async function resizeImageToMatch(imageBuffer: Buffer, width: number, height: number) {
      return sharp(imageBuffer).resize(width, height).toBuffer();
    }
    
    export async function compareImages(url1: string, url2: string): Promise<number> {
      const imageBuffer1 = await downloadImageAsBuffer(url1);
      const imageBuffer2 = await downloadImageAsBuffer(url2);
    
      const metadata1 = await sharp(imageBuffer1).metadata();
      const metadata2 = await sharp(imageBuffer2).metadata();
    
      const targetWidth = Math.min(metadata1.width!, metadata2.width!);
      const targetHeight = Math.min(metadata1.height!, metadata2.height!);
    
      const resizedImageBuffer1 = await resizeImageToMatch(imageBuffer1, targetWidth, targetHeight);
      const resizedImageBuffer2 = await resizeImageToMatch(imageBuffer2, targetWidth, targetHeight);
      return new Promise((resolve) => {
        resemble(resizedImageBuffer1)
          .compareTo(resizedImageBuffer2)
          .onComplete((data) => {
            resolve(Number(data.rawMisMatchPercentage));
          });
      });
    }
    
    ```
    
    ```tsx
    import {
      describe,
      expect, test,
    } from 'vitest';
    import { compareImages } from '../src/util/image';
    
    const sameImageList = [
      ['https://image.bugsm.co.kr/artist/images/200/800696/80069634.jpg?version=20240810002131.0', 'https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/080/145/352/80145352_1723439966398_23_600x600.JPG'],
      ['https://image.bugsm.co.kr/artist/images/200/803473/80347326.jpg?version=20240525002108.0', 'https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/080/957/377/80957377_1716526524390_12_600x600.JPG'],
      ['https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/067/872/918/67872918_1709186664549_22_600x600.JPG', 'https://image.bugsm.co.kr/artist/images/200/800491/80049126.jpg?version=20240301002128.0'],
      ['https://image.bugsm.co.kr/artist/images/200/800291/80029124.jpg?version=20210910002109.0', 'https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/079/946/212/79946212_1631183208798_5_600x600.JPG'],
      ['https://image.bugsm.co.kr/artist/images/200/68/6886.jpg?version=20240223044547.0', 'https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/014/945/137/14945137_1708586411397_21_600x600.JPG'],
    ];
    
    const diffImageList = [
      ['https://image.bugsm.co.kr/artist/images/200/800696/80069634.jpg?version=20240810002131.0', 'https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/080/957/377/80957377_1716526524390_12_600x600.JPG'],
      ['https://image.bugsm.co.kr/artist/images/200/803473/80347326.jpg?version=20240525002108.0', 'https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/080/145/352/80145352_1723439966398_23_600x600.JPG'],
      ['https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/067/872/918/67872918_1709186664549_22_600x600.JPG', 'https://image.genie.co.kr/Y/IMAGE/IMG_ARTIST/079/946/212/79946212_1631183208798_5_600x600.JPG'],
      ['https://image.bugsm.co.kr/artist/images/200/800291/80029124.jpg?version=20210910002109.0', 'https://image.bugsm.co.kr/artist/images/200/800491/80049126.jpg?version=20240301002128.0'],
    ];
    
    describe('Test func compareImages', () => {
      test.each(sameImageList)('Compare two similar images. The raw MisMatch Percentage will be less than 20.', async (a, b) => {
        const rawMisMatchPercentage = await compareImages(a, b);
        expect(rawMisMatchPercentage < 20).toBe(true);
      });
    
      test.each(diffImageList)('Compares two dissimilar images. The raw MisMatch Percentage will be above 80.', async (a, b) => {
        const rawMisMatchPercentage = await compareImages(a, b);
        expect(rawMisMatchPercentage > 80).toBe(true);
      });
    });
    
    ```
    
    ```bash
     image.test.ts (9) 3469ms
       ✓ Test func compareImages (9) 3469ms
         ✓ Compare two similar images. The raw MisMatch Percentage will be less than 20. 938ms
         ✓ Compare two similar images. The raw MisMatch Percentage will be less than 20. 471ms
         ✓ Compare two similar images. The raw MisMatch Percentage will be less than 20. 396ms
         ✓ Compare two similar images. The raw MisMatch Percentage will be less than 20. 321ms
         ✓ Compare two similar images. The raw MisMatch Percentage will be less than 20.
         ✓ Compares two dissimilar images. The raw MisMatch Percentage will be above 80.
         ✓ Compares two dissimilar images. The raw MisMatch Percentage will be above 80. 308ms
         ✓ Compares two dissimilar images. The raw MisMatch Percentage will be above 80. 391ms
         ✓ Compares two dissimilar images. The raw MisMatch Percentage will be above 80.
    ```
    

### map을 이용한 cache 적용

하지만 위방식은 fetch를 해야하기때문에 무거운 비교방식이다. 최후의 보루가 되어야한다(조건부 사용).

너무 많은 이용은 `rate limiter`에 걸릴것이다.  `fetch retry`도있지만 이또한 마지막 수단의 마지막수단이다.

몇번이나 호출될까?

```tsx
 const condition1 = maxArtistSimilarity[1] >= 0.3;
      const condition2 = maxArtistSimilarity[2] === undefined ? true : maxArtistSimilarity[2];
      if (condition1 && condition2) {
        Object.assign(maxArtistSimilarity[0].platforms, { [platformName]: artist });
        await transactionalEntityManager.save(maxArtistSimilarity[0]);
        continue; //  루프 넘어가기
      }

      if (condition1) {
        count += 1;
        console.log(count);
      }
```

여기에 걸리는경우는 총 553회

```tsx
 const condition1 = maxArtistSimilarity[1] >= 0.3;
      const condition2 = maxArtistSimilarity[2] === undefined ? true : maxArtistSimilarity[2];
      if (condition1 && condition2) {
        Object.assign(maxArtistSimilarity[0].platforms, { [platformName]: artist });
        await transactionalEntityManager.save(maxArtistSimilarity[0]);
        continue; //  루프 넘어가기
      }

        count += 1;
        console.log(count);
```

이렇게라면  633회!… 예상외로 크게 차이나진 않는다.

3개의 플랫폼을 생각했을때 200번씩  여기서 동일한결과를 캐시하는 방식을 생각한다면  rate limiter에 걸리진 않을걸로 예상된다. 

뭐 그렇게되더라도 마지막의 마지막수단 `fetch retry` 가있으니.. 라이브러리를 쓸까하다가 간단하게 map으로도 충분할거같다.

- 캐시 로직과 이미지 비교로직을 아래와같이 추가하고
    
    ```tsx
    const imageCache = new Map<string, Buffer>();
    
    async function downloadImageAsBuffer(url: string): Promise<Buffer> {
      if (imageCache.has(url)) {
        return imageCache.get(url)!;
      }
      const response = await axios<Buffer>({
        url,
        responseType: 'arraybuffer',
      });
      const imageBuffer = Buffer.from(response.data);
      imageCache.set(url, imageBuffer); // 캐시에 저장
      return imageBuffer;
    }
    ```
    
    ```tsx
         // 2번 다른 플랫폼인경우
          // artistName 유사도비교
          // debut 비교로직 => conditional
          // artistImage 유사도 비교
          const similarityList: [Artist, number, boolean | undefined][] = existingArtists.map((existingArtist) => {
            const availablePlatform = existingArtist.platforms.melon || existingArtist.platforms.genie || existingArtist.platforms.bugs as ArtistWithAddInfo;
            const result: [Artist, number, boolean | undefined] = [existingArtist, checkSS(availablePlatform.artistName, artist.artistName), undefined];
            if (availablePlatform.debut !== 'missing' && artist.debut !== 'missing') {
              result[2] = availablePlatform.debut.split('.')[0] === artist.debut.split('.')[0];
            }
            return result;
          });
          const maxArtistSimilarity = similarityList.reduce((pre, cur) => (pre[1] > cur[1] ? pre : cur));
    
          const condition1 = maxArtistSimilarity[1] >= 0.3;
          const condition2 = maxArtistSimilarity[2] === undefined ? true : maxArtistSimilarity[2];
          if (condition1 && condition2) {
            Object.assign(maxArtistSimilarity[0].platforms, { [platformName]: artist });
            await transactionalEntityManager.save(maxArtistSimilarity[0]);
            continue; //  루프 넘어가기
          }
          // eslint-disable-next-line no-nested-ternary
          const availablePlatformName = 'bugs' in maxArtistSimilarity[0].platforms ? 'bugs'
            : 'genie' in maxArtistSimilarity[0].platforms ? 'genie'
              : 'melon' as PlatformName;
    
          const rawMisMatchPercentage = await compareImages(artist.artistImage, (maxArtistSimilarity[0].platforms[availablePlatformName] as ArtistWithAddInfo).artistImage);
          if (rawMisMatchPercentage < 20) {
            Object.assign(maxArtistSimilarity[0].platforms, { [platformName]: artist });
            await transactionalEntityManager.save(maxArtistSimilarity[0]);
            continue; //  루프 넘어가기
          }
    ```
    

## Result &  Test code

---

로직을 수정하고 아티스트 통합코드를 돌렸을때 결과는 ….

| command | before | after |
| --- | --- | --- |
| `SELECT COUNT(*) FROM Artist;` | 2735 | 2375 |
| `SELECT artistKeyword, COUNT(*) AS count FROM Artist GROUP BY artistKeyword HAVING COUNT(*) >= 2;` | 470 | 183 |

```tsx
mysql> SELECT COUNT(*) FROM Artist;;
+----------+
| COUNT(*) |
+----------+
|     2357 |
+----------+
1 row in set (0.00 sec)
```

```tsx
mysql> SELECT artistKeyword, COUNT(*) AS count FROM Artist GROUP BY artistKeyword HAVING COUNT(*) >= 2;
+----------------------+-------+
| artistKeyword        | count |
+----------------------+-------+
| 소녀시대             |     3 |
| 백지영               |     2 |
| 현아                 |     2 |
	....
| 옹성우               |     2 |
| 경서                 |     3 |
| yena                 |     2 |
| 선예                 |     3 |
| 김태리               |     2 |
| meghantrainor        |     2 |
+----------------------+-------+
183 rows in set (0.01 sec)
```

track 통합방식에도 해당 로직을 추가적으로 이용했더니 …

| command | before | after |
| --- | --- | --- |
|  `SELECT COUNT(*) FROM track;*` | 10644 | 10291  |
| `SELECT trackKeyword, COUNT(*) AS count  FROM track  GROUP BY trackKeyword  HAVING COUNT(*) >= 2;` | 1245 | 1028 |

```bash
# before

mysql> SELECT COUNT(*) FROM track;;
+----------+
| COUNT(*) |
+----------+
|    10644 |
+----------+
1 row in set (0.02 sec)

mysql> SELECT trackKeyword, COUNT(*) AS count
-> FROM track
-> GROUP BY trackKeyword
-> HAVING COUNT(*) >= 2;
+------------------------------------------------------+-------+
| trackKeyword                                         | count |
+------------------------------------------------------+-------+
| 이사람                                               |     2 |
| dancing queen                                        |     2 |
| 안아보자                                             |     2 |
| 그 사람                                              |     4 |

…

| 마지막 인사                                          |     2 |
| sugarcoat                                            |     2 |
| christian                                            |     2 |
| sos                                                  |     2 |
| 3d                                                   |     2 |
| night dancer                                         |     3 |
+------------------------------------------------------+-------+
1245 rows in set (0.03 sec)
```

```bash
# after
mysql> SELECT COUNT(*) FROM track;
+----------+
| COUNT(*) |
+----------+
|    10291 |
+----------+
1 row in set (0.01 sec)

mysql> SELECT trackKeyword, COUNT(*) AS count  FROM track  GROUP BY trackKeyword  HAVING COUNT(*) >= 2;
+------------------------------------------------------+-------+
| trackKeyword                                         | count |
+------------------------------------------------------+-------+
| dancing queen                                        |     2 |
| 안아보자                                             |     2 |
| 그 사람                                              |     4 |
| yesterday                                            |     5 |
| 되돌리다                                             |     2 |
| 말해봐                                               |     2 |
| don't go                                             |     2 |
| 카페인                                               |     2 |
   ...
| christian                                            |     2 |
| sos                                                  |     2 |
| night dancer                                         |     2 |
+------------------------------------------------------+-------+
1028 rows in set (0.03 sec)

```

많은부분의 디테일을 올렸다고 생각한다.

내가 생각한 로직데로 잘 넣었는지 마지막 검증 =⇒ 테스트코드 확인 ⇒ 😁😆매우 만족 😁😆

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