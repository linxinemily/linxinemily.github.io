---
title: Validate Facebook Limited Login JWT Token in Golang
date: 2024-05-19
tags:
- Golang
- Third-Party Integration
---

## 前提

Facebook ios sdk 在 17.0.0 版本後，要求必須實作 Limited Login，並使用 JWT token 進行驗證，否則在某些情況下會有非預期錯誤([ref](https://github.com/facebook/facebook-ios-sdk/issues/2365))。

流程為：

 client 端（ios App） 會將以下資訊：

- 登入嘗試要求的權限
- 追蹤偏好設定
- nonce（隨機生產的亂數，每次請求都須不同，用來做後續的驗證）

帶入請求，向 FB 取得用戶資料與一組 JWT token。（實作範例可參考[官方文件](https://developers.facebook.com/docs/facebook-login/limited-login/ios)）

接著 client 會將 token 傳給我們的後端 server，後端需要驗證 token 是否合法。

重點就在於後端驗證 token 這段，[官方文件](https://developers.facebook.com/docs/facebook-login/limited-login/token/validating)非常簡明概要，並沒有附上實作範例。所以特此紀錄一下~~踩坑過程~~。

## 如何驗證 JWT Token

首先，[文件](https://developers.facebook.com/docs/facebook-login/limited-login/token/validating)要我們確認以下三件事：

##### 1. [That the JWT is well formed](https://developers.facebook.com/docs/facebook-login/limited-login/token/validating#jwt-well-formed)

token 須為合法的 JWT token。也就是包含一組用 Base64Url-encoded 的 header, payload, signature，每個部分解碼後都必須為合法的 json 格式。

##### 2. [The signature](https://developers.facebook.com/docs/facebook-login/limited-login/token/validating#signature)

驗證 signature 部分是否合法：

1. 藉由呼叫 [JWKS endpoint](https://developers.facebook.com/docs/facebook-login/limited-login/token/#jwks) 取得一組 public key
    
    > 打 [JWKS endpoint](https://developers.facebook.com/docs/facebook-login/limited-login/token/#jwks) 會得到一個 json 物件，回傳多組 public key：
    > 
    > 
    > ```json
    > "keys": [
    >       {
    >          "kid": "d458ab5237807dc6718901e522cebcd8e8157791",
    >          "kty": "RSA",
    >          "alg": "RS256",
    >          "use": "sig",
    >          "n": "uPyWMhNfNsO9EtiraYI0tr78vnkiJmzsmAAUd8hLHF5vPXDn683aQKZQ2Ny5lObigNmbHI5tt5y0o5m0RuZjJTj081uWm7Z901boO-p4VLwEONzjh4vTp2ZQ7aMjo17kMBzInHqz9iruWeB94dEu_LKYdQnDI6rweD_-chWWTR4mc7xbeaNozLHYzjEisSrIM3xIry2lZv5Mh334ZoahcTXGouFtU2XV_HvStXthwhoAtizQK7s2yJlBz8qlQK2lFNojRzd95f2bkynRnIvcpoF-qHZbOBTCIf-6TLp23qShs-XvbCkwHMhzvCPxcuZx3GNfCQkyTxeM5IGIMlWZ8w",
    >          "e": "AQAB"
    >       },
    >       ...
    > ```
    > 
    > 而要使用哪一組 key，需要用解碼的 header 當中所帶的一個屬性 `kid` 去比對，找出相同 `kid`  的那把 key。
    > 
2. 使用解碼後的 header 當中指定的演算法(欄位： `alg`)以及上述的 public key 對 Base64Url-encoded 的 header 跟 payload 的組合(`Base64url-encoded header + "." + Base64url-encoded payload`) 進行簽名。
3. 將簽名處理的結果進行 Base64url 編碼，檢查是否和 signature 相等，如果相等代表簽名有效，JWT 合法

##### 3. [The standard claims](https://developers.facebook.com/docs/facebook-login/limited-login/token/validating#standard-claims)

檢查解碼後的 payload 幾個屬性值是否符合預期：

1. 檢查 token 有無過期（欄位： `exp`）
2. Token issuer 是否正確（欄位： `iss` ）
    
    > 文件上沒有說明 issuer 的值為何（照理來說會是 Facebook 的某個網域或端點），後來是在某篇討論串當中找到，在[限制登入的 OIDC 權杖](https://developers.facebook.com/docs/facebook-login/limited-login/token/)的 **OIDC 端點 > [探索端點](https://limited.facebook.com/.well-known/openid-configuration/)** 裡面：
    > 
    > 
    > ```json
    > {
    >     "issuer": "https://www.facebook.com",
    >     //...
    > }
    > ```
    > 
3. Audience 是否與 FB App ID 相同 （欄位： `aud` ）
    
	App ID 可以到 Facebook developer 後台查看
    
4. Nonce 是否正確（欄位： `nonce` ）

   就是上面前端傳給 FB 產 token 時帶的亂數，前端在發 request 給後端時需一併攜帶過來。（後端才能驗證 payload 當中的 nonce 是否與之相同）

## 實作

以上解析 JWT token 與驗證 signature 的部分都有現成的 JWT 套件可以替我們完成，在這邊使用 golang 搭配以下兩個套件來實作：

- [golang-jwt/jwt/v5](https://pkg.go.dev/github.com/golang-jwt/jwt/v5)：解析 JWT token 與驗證 signature
- [lestrrat-go/jwx/jwk](https://github.com/lestrrat-go/jwx)：解析 JWKS public key

首先定義 structs 接收需要的資料：

```go
type (
	Auth struct {
		JWTToken     string
		Nonce        string
		APIHost      string
		AppID        string
		JWKSEndpoint string
		Client       *http.Client
	}

	// 檢查 token 有無過期、驗證 issuer 與 audience 都是屬於 JWT 規範中定義的一些常見聲明
	// 可以使用 jwt.RegisteredClaims 即可
	// 但因為我們有攜帶 `Nonce`，為 FB 自定義的部分，所以需要自訂 Claim
	ClientClaims struct {
		jwt.RegisteredClaims
		Nonce string `json:"nonce"`
	}
)

func NewFBAuth(jwtToken, nonce string) *Auth {
	return &Auth{
		JWTToken:     jwtToken,
		Nonce:        nonce,
		//屬於敏感資訊，以 env config 形式取得
		APIHost:      config.Config.FacebookAuth.APIHost, // 預期的 issuer 值
		AppID:        config.Config.FacebookAuth.AppID, // 預期的 audience 值
		JWKSEndpoint: config.Config.FacebookAuth.JWKSEndpoint,
		Client:       &http.Client{Timeout: 30 * time.Second},
	}
}
```

接著實作驗證方法的部分：

```go
func (a *Auth) ValidateJWTToken() error {
	set, err := a.fetchPublicKeySet() // 取得 JWKS Public key
	if err != nil {
		return err
	}

	token, err := jwt.NewParser( // 使用套件內建的 validator 驗證標準聲明的欄位
		jwt.WithExpirationRequired(), // 檢查 token 有無過期
		jwt.WithIssuer(a.APIHost), // Token issuer 是否正確（等於 a.APIHost）
		jwt.WithAudience(a.AppID), // Audience 是否正確（等於 a.AppID）
	).ParseWithClaims(a.JWTToken, &ClientClaims{}, func(token *jwt.Token) (interface{}, error) {
		// 取得 header 的 kid 值，用來尋找 JWKS 端點回傳 json 當中匹配的 pub key
		keyID, ok := token.Header["kid"].(string)
		if !ok {
			return nil, errors.Str("expecting JWT header to have string kid")
		}

		keys := set.LookupKeyID(keyID)
		if len(keys) == 0 {
			return nil, errors.Str("can not find kid")
		}

		var key interface{}
		if err := keys[0].Raw(&key); err != nil {
			return nil, err
		}

		return key, nil
	})
	if err != nil {
		return err
	} else if claims, ok := token.Claims.(*ClientClaims); ok {
		// 由於 Nonce 為自定義的聲明屬性，需要額外手動驗證
		if claims.Nonce != a.Nonce {
			return errors.Errorf("nonce mismatch: %s != %s", claims.Nonce, a.Nonce)
		}
	} else {
		return errors.Errorf("expecting *ClientClaims, got %T", token.Claims)
	}

	return nil
}

func (a *Auth) fetchPublicKeySet() (*jwk.Set, error) {
	resp, err := a.Client.Get(a.JWKSEndpoint)
	if err != nil {
		return nil, err
	}
	respBody, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	return jwk.ParseBytes(respBody)
}
```

驗證成功即可向前端回傳使用者登入成功。

## 碎碎念

猜測是因為後端語言太多，各語言也都有自己的 jwt 套件，所以官方才沒有放驗證 Token 的實作範例，但像上面提到的 kid、issuer 應該可以說明更清楚一點。不過也算是藉機學習了，~~感謝 Facebook。（？）~~