# OAuth2 인증 API 활용 커스텀 커넥터 만들기 #

파워 플랫폼 커스텀 커넥터에 연동시키기 위한 API를 개발합니다. 이 API는 애저 API 관리자에서 프록시 형태로 지원하는 OAuth2 인증을 구현합니다. 아래 순서대로 따라해 보세요.

- [OAuth2 인증 API 활용 커스텀 커넥터 만들기](#oauth2-인증-api-활용-커스텀-커넥터-만들기)
  - [1. OAuth2 인증용 앱 등록하기](#1-oauth2-인증용-앱-등록하기)
  - [2. API 앱 개발하기](#2-api-앱-개발하기)
  - [3. GitHub 액션 연동후 자동 배포하기](#3-github-액션-연동후-자동-배포하기)
  - [4. API 관리자 연동하기](#4-api-관리자-연동하기)
  - [5. API 관리자 OAuth2 등록하기](#5-api-관리자-oauth2-등록하기)
  - [6. 파워 플랫폼 커스텀 커넥터 생성하기](#6-파워-플랫폼-커스텀-커넥터-생성하기)
  - [7. 파워 앱과 파워 오토메이트에서 커스텀 커넥터 사용하기](#7-파워-앱과-파워-오토메이트에서-커스텀-커넥터-사용하기)
    - [파워 오토메이트](#파워-오토메이트)
    - [파워 앱](#파워-앱)

## 1. OAuth2 인증용 앱 등록하기 ##

이번에 사용하려는 API는 OAuth2 인증을 통한 액세스 토큰이 필요합니다. 이 액세스 토큰을 발급받기 위해서는 먼저 [OAuth2 인증용 앱][az ad oauth2]을 [애저 액티브 디렉토리][az ad]에 등록해야 합니다. 아래 순서대로 따라해 보세요.

1. 애저 포털 상단의 검색창에 `azure active directory`를 검색합니다.

    ![애저 포털에서 액티브 디렉토리 검색하기][image01]

2. 액티브 디렉토리 화면에서 **앱 등록** 블레이드로 이동합니다. 그러면 이미 [애저 Dev CLI 이용해서 애저 인스턴스 만들기](./1-azd.md) 세션에서 애저 CLI를 통해 **spn-gppb{{랜덤숫자}}**라는 이름으로 만들었던 앱이 하나 있는 것이 보입니다. 여기서 `{{랜덤숫자}}`는 앞서 `echo $RANDOM`으로 생성한 숫자를 가리킵니다.

    ![등록한 앱 확인][image02]

3. 위의 앱은 GitHub 액션 워크플로우 용도로 만든 것이므로 사용하지 않고, API 관리자 용도로 하나 다시 만들겠습니다. **+ 새 등록** 메뉴를 클릭합니다.

    ![새 앱 등록][image03]

4. 애플리케이션 등록 화면에서 아래와 같이 각각의 필드에 입력하고 **등록** 버튼을 클릭합니. 여기서 `{{랜덤숫자}}`는 앞서 `echo $RANDOM`으로 생성한 숫자를 가리킵니다.

   - **이름**: `spn-gppb{{랜덤숫자}}-apim`
   - **지원되는 계정 유형**: `이 조직 디렉터리의 계정만(Microsoft만 - 단일 테넌트)` 선택
   - **리디렉션 URI(선택 사항)**: `웹` ➡️ `https://oauth.pstmn.io/v1/browser-callback`

     > **NOTE**: 여기서 `https://oauth.pstmn.io/v1/browser-callback`는 이후 잠깐 테스트를 해 볼 [포스트맨](https://www.postman.com)의 콜백 URL입니다.

    ![애플리케이션 등록][image04]

5. 앱 등록 이후 **개요** 블레이드에서 **애플리케이션(클라이언트) ID** 값과 **디렉터리(테넌트) ID** 값을 복사합니다.

    ![애플리케이션 등록 확인 #1][image05]

6. 방금 등록한 애플리케이션의 **인증** 블레이드를 클릭해서 등록한 내용을 확인합니다.

    ![애플리케이션 등록 확인 #2][image06]

7. **인증서 및 암호** 블레이드로 이동해서 새 클라이언트 암호를 등록해야 합니다. **클라이언트 비밀** ➡️ **+ 새 클라이언트 암호** 메뉴를 클릭한 후 아래와 같이 입력합니다. 이후 **추가** 버튼을 클릭합니다.

   - **설명**: `apim` 입력
   - **만료 시간**: `Recommended: 180d dys (6 months)` 선택

    ![클라이언트 시크릿 추가][image07]

8. 아래 그림과 같이 클라이언트 시크릿이 만들어졌습니다. 이 값을 복사해서 어딘가에 저장해 둡니다. 이 화면을 벗어나는 순간 더이상 확인할 방법이 없으니 주의하세요!

    ![클라이언트 시크릿 확인][image08]

   이제 아래 값들을 복사해 둔 것을 반드시 기억하세요.

   - **디렉터리(테넌트) ID**
   - **애플리케이션(클라이언트) ID**
   - **클라이언트 암호(시크릿)**

9. **API 사용 권한** 블레이드로 이동해서 아래 그림과 같이 **API/권한 이름** 항목에 `Microsoft Graph/User.Read` 권한이 추가되었는지 확인하세요.

    ![API 사용 권한 확인][image09]

   만약 위와 같은 내용이 보이지 않는다면 **🔄 새로 고침** 메뉴를 클릭해 보세요. 만약 그래도 보이지 않는다면 아래와 같이 **➕ 권한 추가** 메뉴를 클릭해서 추가해야 합니다. **Microsoft Graph** ➡️ **위임된 권한** ➡️ **User** ➡️ **User.Read**를 선택한 후 **권한 추가** 버튼을 클릭합니다.

    ![API 사용 권한 추가][image10]

애저 액티브 디렉토리에 OAuth2 인증을 위한 애플리케이션을 등록했습니다.

## 2. API 앱 개발하기 ##

이미 최소한의 작동을 하는 API 앱이 [애저 펑션][az fncapp]으로 만들어져 있습니다. 이 API 앱을 애저 API 관리자에 연동하기 위한 작업으로 OpenAPI 문서 자동 생성 도구를 추가해 보겠습니다.

1. 아래 명령어를 통해 API 앱을 생성합니다.

    ```bash
    unzip ./custom-connectors-in-a-day/AuthCodeAuthApp.zip -d ./custom-connectors-in-a-day/src
    ```

2. 아래 명령어를 실행시켜 방금 생성한 API 앱을 솔루션에 연결시킵니다.

    ```bash
    pushd ./custom-connectors-in-a-day \
        && dotnet sln add ./src/AuthCodeAuthApp -s src \
        && popd
    ```

3. 아래 명령어를 실행시켜 GitHub 코드스페이스에서 API 앱을 실행시킬 수 있게끔 `local.settings.json` 파일을 생성합니다.

    ```bash
    pwsh -c "Invoke-RestMethod https://aka.ms/azfunc-openapi/add-codespaces.ps1 | Invoke-Expression"
    ```

4. 아래 명령어를 통해 API 앱을 실행시킵니다.

    ```bash
    pushd ./custom-connectors-in-a-day/src/AuthCodeAuthApp \
        && dotnet restore && dotnet build \
        && func start
    ```

5. 아래 팝업창이 나타나면 **Open in Browser** 버튼을 클릭합니다.

    ![새 창에서 API 열기 #1][image11]

6. 아래와 같은 화면이 나타나면 API 앱이 성공적으로 작동하는 것입니다.

    ![애저 펑션 앱 실행 결과 #1][image12]

7. 이제 주소창의 URL 맨 뒤에 `/api/profile`을 붙인후 아래와 같은 결과가 나오는지 확인합니다.

    ![애저 펑션 앱 API 호출 결과][image13]

8. 이제 터미널에서 `Control + C` 키를 눌러 애저 펑션 앱을 종료합니다.

9. 아래 명령어를 실행시켜 리포지토리의 루트 디렉토리로 돌아옵니다.

    ```bash
    popd
    ```

10. `custom-connectors-in-a-day/src/AuthCodeAuthApp/AuthCodeAuthApp.csproj` 파일을 열어 아래 부분의 주석을 제거합니다.

    ```xml
    ...
    <ItemGroup>
      <PackageReference Include="Microsoft.Azure.Functions.Extensions" Version="1.1.0" />
      <PackageReference Include="Microsoft.Extensions.Configuration.UserSecrets" Version="6.0.1" />
      <PackageReference Include="Microsoft.Extensions.Http" Version="6.0.0" />
      <PackageReference Include="Microsoft.NET.Sdk.Functions" Version="4.1.3" />
  
      <!-- ⬇️⬇️⬇️ 아래의 코드 주석을 풀어주세요 ⬇️⬇️⬇️ -->
      <PackageReference Include="Microsoft.Azure.WebJobs.Extensions.OpenApi" Version="1.*" />
      <!-- ⬆️⬆️⬆️ 위의 코드 주석을 풀어주세요 ⬆️⬆️⬆️ -->
    </ItemGroup>
    ...
    ```

11. `custom-connectors-in-a-day/src/AuthCodeAuthApp/Startup.cs` 파일을 열어 아래 부분의 주석을 제거합니다.

    ```csharp
    ...
    // ⬇️⬇️⬇️ 아래의 코드 주석을 풀어주세요 ⬇️⬇️⬇️
    using Microsoft.Azure.WebJobs.Extensions.OpenApi.Configurations.AppSettings.Extensions;
    using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Abstractions;
    using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Configurations;
    using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Enums;
    using Microsoft.OpenApi.Models;
    // ⬆️⬆️⬆️ 위의 코드 주석을 풀어주세요 ⬆️⬆️⬆️
    ...
    private static void ConfigureAppSettings(IServiceCollection services)
    {
        // ⬇️⬇️⬇️ 아래의 코드 주석을 막아주세요 ⬇️⬇️⬇️
        // var settings = new GraphSettings();
        // services.BuildServiceProvider()
        //         .GetService<IConfiguration>()
        //         .GetSection(GraphSettings.Name)
        //         .Bind(settings);
        // services.AddSingleton(settings);
        // ⬆️⬆️⬆️ 위의 코드 주석을 막아주세요 ⬆️⬆️⬆️

        // ⬇️⬇️⬇️ 아래의 코드 주석을 풀어주세요 ⬇️⬇️⬇️
        var settings = services.BuildServiceProvider()
                                .GetService<IConfiguration>()
                                .Get<GraphSettings>(GraphSettings.Name);
        services.AddSingleton(settings);

        var options = new DefaultOpenApiConfigurationOptions()
        {
            OpenApiVersion = OpenApiVersionType.V3,
            Info = new OpenApiInfo()
            {
                Version = "1.0.0",
                Title = "API AuthN'd by Authorization Code Auth",
                Description = "This is the API authN'd by Authorization Code Auth."
            }
        };

        var codespaces = bool.TryParse(Environment.GetEnvironmentVariable("OpenApi__RunOnCodespaces"), out var isCodespaces) && isCodespaces;
        if (codespaces)
        {
            options.IncludeRequestingHostName = false;
        }

        services.AddSingleton<IOpenApiConfigurationOptions>(options);
    }
    ...
    ```

12. `custom-connectors-in-a-day/src/AuthCodeAuthApp/AuthCodeAuthHttpTrigger.cs` 파일을 열어 아래 부분의 주석을 제거합니다.

    ```csharp
    ...
    // ⬇️⬇️⬇️ 아래의 코드 주석을 풀어주세요 ⬇️⬇️⬇️
    using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Attributes;
    using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Enums;
    using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Extensions;
    using Microsoft.OpenApi.Models;
    // ⬆️⬆️⬆️ 위의 코드 주석을 풀어주세요 ⬆️⬆️⬆️
    ...
    [FunctionName(nameof(ApiKeyAuthHttpTrigger.GetGreeting))]

    // ⬇️⬇️⬇️ 아래의 코드 주석을 풀어주세요 ⬇️⬇️⬇️
    [OpenApiOperation(operationId: "Profile", tags: new[] { "profile" })]
    [OpenApiSecurity("bearer_auth", SecuritySchemeType.Http, Scheme = OpenApiSecuritySchemeType.Bearer, BearerFormat = "JWT")]
    [OpenApiResponseWithBody(statusCode: HttpStatusCode.OK, contentType: "application/json", bodyType: typeof(GraphUser), Description = "The OK response")]
    [OpenApiResponseWithBody(statusCode: HttpStatusCode.BadRequest, contentType: "text/plain", bodyType: typeof(string), Description = "The bad request response")]
    // ⬆️⬆️⬆️ 위의 코드 주석을 풀어주세요 ⬆️⬆️⬆️

    public async Task<IActionResult> GetProfile(
    ...
    ```

13. 아래 명령어를 통해 API 앱을 실행시킵니다.

    ```bash
    pushd ./custom-connectors-in-a-day/src/AuthCodeAuthApp \
        && dotnet build \
        && func start
    ```

14. 아래 팝업창이 나타나면 **Open in Browser** 버튼을 클릭합니다.

    ![새 창에서 API 열기 #2][image14]

15. 아래와 같은 화면이 나타나면 API 앱이 성공적으로 작동하는 것입니다.

    ![애저 펑션 앱 실행 결과 #2][image15]

16. 이제 주소창의 URL 맨 뒤에 `/api/swagger/ui`을 붙인후 아래와 같은 화면이 나오는지 확인합니다.

    ![애저 펑션 Swagger UI][image16]

17. 위 Swagger UI 화면에서 화살표가 가리키는 링크를 클릭합니다.

    ![애저 펑션 Swagger UI에서 swagger.json 문서 링크 클릭][image17]

18. 아래 그림과 같이 `swagger.json`라는 이름으로 OpenAPI 문서가 보이는지 확인합니다.

    ![애저 펑션 OpenAPI 문서 출력][image18]

19. 이제 터미널에서 `Control + C` 키를 눌러 애저 펑션 앱을 종료합니다.

20. 아래 명령어를 실행시켜 리포지토리의 루트 디렉토리로 돌아옵니다.

    ```bash
    popd
    ```

[애저 펑션][az fncapp]을 이용한 OAuth2 인증용 API 앱에 OpenAPI 기능을 추가하는 과정이 끝났습니다.


## 3. GitHub 액션 연동후 자동 배포하기 ##

앞서 개발한 API 앱을 GitHub 액션 워크플로우를 이용해 애저에 배포합니다. 아래 순서대로 따라해 보세요.

1. `custom-connectors-in-a-day/infra/gha-matrix.json` 파일을 열어 아래와 같이 수정합니다.

    ```jsonc
    [
      {
        "name": "apikeyauth",
        "suffix": "api-key-auth",
        "path": "ApiKeyAuthApp",
        "nv": "API_KEY_AUTH"
      },
      {
        "name": "basicauth",
        "suffix": "basic-auth",
        "path": "BasicAuthApp",
        "nv": "BASIC_AUTH"
      }, // 👈 쉼표 잊지 마세요

      // ⬇️⬇️⬇️ 아래 JSON 개체를 추가하세요 ⬇️⬇️⬇️
      {
        "name": "authcodeauth",
        "suffix": "auth-code-auth",
        "path": "AuthCodeAuthApp",
        "nv": "AUTH_CODE_AUTH"
      }
      // ⬆️⬆️⬆️ 위 JSON 개체를 추가하세요 ⬆️⬆️⬆️
    ]
    ```

2. 변경한 API 앱을 깃헙에 커밋합니다.

    ```bash
    git add . \
        && git commit -m "OAuth2 인증 앱 수정" \
        && git push origin
    ```

3. 아래와 같이 GitHub 액션 워크플로우가 자동으로 실행되는 것을 확인합니다.

    ![GitHub 액션 워크플로우 실행중][image09]

4. 아래와 같이 모든 GitHub 액션 워크플로우가 성공적으로 실행된 것을 확인합니다.

    ![GitHub 액션 워크플로우 실행 완료][image10]

5. 웹브라우저 주소창에 방금 배포한 API 앱의 주소를 입력하고 Swagger UI 화면이 나오는지 확인합니다. `{{랜덤숫자}}`는 앞서 `echo $RANDOM`으로 생성한 숫자를 가리킵니다.

    ```bash
    https://fncapp-gppb{{랜덤숫자}}-auth-code-auth.azurewebsites.net/api/swagger/ui
    ```

    ![배포된 API 앱 Swagger UI][image11]

6. 아래 그림과 같이 **Authorize** 버튼을 클릭합니다.

    ![인증 버튼 클릭][image12]

7. 앞서 생성한 Atlassian 계정의 이메일 주소와 API 토큰을 입력한 후 **Authorize** 버튼을 클릭합니다.

    ![인증 정보 입력][image13]

8. 아래와 같이 자물쇠 모양이 잠긴 것을 확인한 후 **Try it out** 버튼을 클릭합니다.

    ![API 테스트][image14]

9. 이후 **Execute** 버튼을 클릭하면 아래와 같이 `401 Unauthorized` 에러가 나오는 것을 확인하세요.

    ![API 테스트 결과][image15]

10. 아래와 같이 요청 헤더 부분의 `Basic ***`으로 시작하는 문자열을 복사해서 보관합니다. 이후 API 관리자 연동후 테스트 용도로 사용할 인증 키 값입니다.

    ![Basic 인증 키 값 복사][image16]

11. 이제 아래 그림의 화살표가 가리키는 링크를 클릭해서 OpenAPI 문서를 표시합니다.

    ![배포된 API 앱 OpenAPI 문서 링크][image17]

12. OpenAPI 문서가 표시되는 것을 확인합니다.

    ![배포된 API 앱 OpenAPI 문서 생성][image18]

13. 이 OpenAPI 문서의 주소를 복사해 둡니다. 주소는 대략 아래와 같은 형식입니다. `{{랜덤숫자}}`는 앞서 `echo $RANDOM`으로 생성한 숫자를 가리킵니다.

    ```text
    https://fncapp-gppb{{랜덤숫자}}-basic-auth.azurewebsites.net/api/swagger.json
    ```

14. 아래 명령어를 실행시켜 애저 펑션의 환경 변수 값을 업데이트합니다. `{{ATLASSIAN_INSTANCE_NAME}}` 값을 앞서 기록해 둔 내 Atlassian 인스턴스 이름으로 대체합니다.

    ```bash
    ATLASSIAN_INSTANCE_NAME="{{ATLASSIAN_INSTANCE_NAME}}"
    AZURE_ENV_NAME="gppb{{랜덤숫자}}"
    resgrp="rg-$AZURE_ENV_NAME"
    fncapp="fncapp-$AZURE_ENV_NAME-basic-auth"

    az functionapp config appsettings set \
        -g $resgrp \
        -n $fncapp \
        --settings Atlassian__InstanceName=$ATLASSIAN_INSTANCE_NAME
    ```

[애저 펑션][az fncapp]을 이용한 OAuth2 인증용 API 앱 배포가 끝났습니다.



## 4. API 관리자 연동하기 ##

TBD


## 5. API 관리자 OAuth2 등록하기 ##

TBD


## 6. 파워 플랫폼 커스텀 커넥터 생성하기 ##

TBD


## 7. 파워 앱과 파워 오토메이트에서 커스텀 커넥터 사용하기 ##

TBD


### 파워 오토메이트 ###

TBD


### 파워 앱 ###

TBD



파워 앱에서 파워 오토메이트를 통해 커스텀 커넥터를 연결하고 API를 호출해 봤습니다.

---

여기까지 해서 Basic 인증을 통한 파워 플랫폼 커스텀 커넥터를 만들고, 이를 파워 앱과 파워 오토메이트에서 활용해 봤습니다.

- [애저 액티브 디렉토리 OAuth2 &ndash; 인증 코드 플로우][az ad oauth2 flow authcode]
- [파워 앱 더 알아보기][pp apps]
- [파워 오토메이트 더 알아보기][pp auto]

---

- 이전 세션: [Basic 인증 API 개발, 애저 API 관리자와 통합, 그리고 커스텀 커넥터 만들기](./3-basic-auth.md)


[image01]: ./images/session04-image01.png
[image02]: ./images/session04-image02.png
[image03]: ./images/session04-image03.png
[image04]: ./images/session04-image04.png
[image05]: ./images/session04-image05.png
[image06]: ./images/session04-image06.png
[image07]: ./images/session04-image07.png
[image08]: ./images/session04-image08.png
[image09]: ./images/session04-image09.png
[image10]: ./images/session04-image10.png
[image11]: ./images/session04-image11.png
[image12]: ./images/session04-image12.png
[image13]: ./images/session04-image13.png
[image14]: ./images/session04-image14.png
[image15]: ./images/session04-image15.png
[image16]: ./images/session04-image16.png
[image17]: ./images/session04-image17.png
[image18]: ./images/session04-image18.png
[image19]: ./images/session04-image19.png
[image20]: ./images/session04-image20.png
[image21]: ./images/session04-image21.png
[image22]: ./images/session04-image22.png
[image23]: ./images/session04-image23.png
[image24]: ./images/session04-image24.png
[image25]: ./images/session04-image25.png
[image26]: ./images/session04-image26.png
[image27]: ./images/session04-image27.png
[image28]: ./images/session04-image28.png
[image29]: ./images/session04-image29.png
[image30]: ./images/session04-image30.png
[image31]: ./images/session04-image31.png
[image32]: ./images/session04-image32.png
[image33]: ./images/session04-image33.png
[image34]: ./images/session04-image34.png
[image35]: ./images/session04-image35.png
[image36]: ./images/session04-image36.png
[image37]: ./images/session04-image37.png
[image38]: ./images/session04-image38.png
[image39]: ./images/session04-image39.png
[image40]: ./images/session04-image40.png
[image41]: ./images/session04-image41.png
[image42]: ./images/session04-image42.png
[image43]: ./images/session04-image43.png
[image44]: ./images/session04-image44.png
[image45]: ./images/session04-image45.png
[image46]: ./images/session04-image46.png
[image47]: ./images/session04-image47.png
[image48]: ./images/session04-image48.png
[image49]: ./images/session04-image49.png
[image50]: ./images/session04-image50.png
[image51]: ./images/session04-image51.png
[image52]: ./images/session04-image52.png
[image53]: ./images/session04-image53.png
[image54]: ./images/session04-image54.png


[az ad]: https://learn.microsoft.com/ko-kr/azure/active-directory/fundamentals/active-directory-whatis?WT.mc_id=dotnet-87051-juyoo
[az ad oauth2]: https://learn.microsoft.com/ko-kr/azure/active-directory/fundamentals/auth-oauth2?WT.mc_id=dotnet-87051-juyoo
[az ad oauth2 flow authcode]: https://learn.microsoft.com/ko-kr/azure/active-directory/develop/v2-oauth2-auth-code-flow?WT.mc_id=dotnet-87051-juyoo

[az fncapp]: https://learn.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-87051-juyoo

[az apim]: https://learn.microsoft.com/ko-kr/azure/api-management/api-management-key-concepts?WT.mc_id=dotnet-87051-juyoo

[pp apps]: https://learn.microsoft.com/ko-kr/power-apps/powerapps-overview?WT.mc_id=dotnet-87051-juyoo
[pp auto]: https://learn.microsoft.com/ko-kr/power-automate/getting-started?WT.mc_id=dotnet-87051-juyoo
[pp cuscon]: https://learn.microsoft.com/ko-kr/connectors/custom-connectors/?WT.mc_id=dotnet-87051-juyoo
