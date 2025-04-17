# Azure-APIManagement

## 필수 헤더키 추가
```xml
<!--
    - Policies are applied in the order they appear.
    - Position <base/> inside a section to inherit policies from the outer scope.
    - Comments within policies are not preserved.
-->
<!-- Add policies as children to the <inbound>, <outbound>, <backend>, and <on-error> elements -->
<policies>
    <inbound>
        <base />
        <!-- 'Test-Key' 필수 헤더 키 검증로직 -->
        <choose>
            <!-- <when condition="@( !context.Request.Headers.ContainsKey("TEST-KEY"))"> -->
            <when condition="@( !context.Request.Headers.ContainsKey("TEST-KEY") || context.Request.Headers.GetValueOrDefault("TEST-KEY","") != "test")">
                <return-response>
                    <set-status code="400" reason="Bad Request (KMS)" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>{"error": "필수 헤더 누락 테스트. TEST-KEY값이 옳바르지 않거나 없음"}</set-body>
                </return-response>
            </when>
            <otherwise />
        </choose>

        <!-- Azure OpenAI BackEnd 연동-->
        <set-backend-service id="apim-generated-policy" backend-id="umla-mkc-oai-prd" />
        
        <!-- 
            # Azure OpenAI Key 등록 - Azure OpenAI를 API방식으로 통신
            <set-header name="api-key" exists-action="override">
                <value>{{umla-mkc-oai-prd-key}}</value>
            </set-header>
        -->
        
        <!-- # Azure OpenAI 관리ID 적용 -->
        <authentication-managed-identity resource="https://cognitiveservices.azure.com" />
        
        <!-- 
            PTU관련 접속 제한 설정
            renewal-period : 제한 시간 계산(초단위, 15 : 15초마다 카운터 초기화)
            calls : renewal-period동안 허용되는 최대 호출 횟수(3 : 3번 호출 허용)
            
            <rate-limit-by-key calls="3" renewal-period="15" counter-key="@(context.Subscription.Id)" />
            - calls="3" renewal-period="15" : 각 구독마다 별도로 15초에 3번 호출 허용
            - counter-key : 제한을 적용할 기준 키
                - Subscription.Id : 구독 id를 기준
                - GetValueOrDefault("Ocp-Apim-Subscription-Key","anonymous") : APIM 구독 키를 기준
                    - anonymous : 기본적으로 apim키가 없으면 접근불가(401)
                    - 예외 : Subscription key not required옵션 사용경우 apim키가 없어도 통과
        -->
        <rate-limit-by-key calls="3" renewal-period="15" counter-key="@(context.Request.Headers.GetValueOrDefault("Ocp-Apim-Subscription-Key","anonymous"))" />
    
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

## APIManagement Policies If문 Test
```xml
<policies>
    <inbound>
        <base />
        <!-- Azure OpenAI BackEnd 연동-->
        <set-backend-service id="apim-generated-policy" backend-id="kms-test-aoai-prd-backend" />
        <!-- # Azure OpenAI Key를 통한 연결 - Azure OpenAI를 API방식으로 통신 -->
        <set-header name="api-key" exists-action="override">
            <value>{{kms-test-aoai-prd-key}}</value>
        </set-header>
        <!-- 'Test-Key' 필수 헤더 키 검증로직 -->
        <choose>
            <!-- <when condition="@( !context.Request.Headers.ContainsKey("TEST-KEY"))"> -->
            <when condition="@( !context.Request.Headers.ContainsKey("TEST-KEY"))">
                <return-response>
                    <set-status code="400" reason="Bad Request (KMS)" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>{"error": "TEST-KEY값이 없음"}</set-body>
                </return-response>
            </when>
            <when condition="@( 
                    context.Request.Headers.ContainsKey("TEST-KEY") && 
                    context.Request.Headers.GetValueOrDefault("TEST-KEY","") != "test"
                    )">
                <return-response>
                    <set-status code="400" reason="Bad Request (KMS)" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>{"error": "TEST-KEY값이 잘못됨"}</set-body>
                </return-response>
            </when>
            <when condition="@( 
                    context.Request.Headers.ContainsKey("TEST-KEY") &&
                    context.Request.Headers.ContainsKey("CS") &&
                    context.Request.Headers.GetValueOrDefault("CS","") == "1"
                    )">
                <return-response>
                    <set-status code="400" reason="Bad Request (KMS)" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>{"error": "Content Safety 위반 1단계"}</set-body>
                </return-response>
            </when>
            <when condition="@( 
                    context.Request.Headers.ContainsKey("TEST-KEY") &&
                    context.Request.Headers.ContainsKey("CS") &&
                    context.Request.Headers.GetValueOrDefault("CS","") == "2"
                    )">
                <return-response>
                    <set-status code="400" reason="Bad Request (KMS)" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>{"error": "Content Safety 위반 2단계"}</set-body>
                </return-response>
            </when>
            <otherwise />
        </choose>
        <!-- 
            PTU관련 접속 제한 설정
            renewal-period : 제한 시간 계산(초단위, 15 : 15초마다 카운터 초기화)
            calls : renewal-period동안 허용되는 최대 호출 횟수(3 : 3번 호출 허용)
            
            <rate-limit-by-key calls="3" renewal-period="15" counter-key="@(context.Subscription.Id)" />
            - calls="3" renewal-period="15" : 각 구독마다 별도로 15초에 3번 호출 허용
            - counter-key : 제한을 적용할 기준 키
                - Subscription.Id : 구독 id를 기준
                - GetValueOrDefault("Ocp-Apim-Subscription-Key","anonymous") : APIM 구독 키를 기준
                    - anonymous : 기본적으로 apim키가 없으면 접근불가(401)
                    - 예외 : Subscription key not required옵션 사용경우 apim키가 없어도 통과
        -->
        <rate-limit-by-key calls="3" renewal-period="15" counter-key="@(context.Request.Headers.GetValueOrDefault("Ocp-Apim-Subscription-Key","anonymous"))" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

## APIManagement 중첩 if문
```xml
<policies>
    <inbound>
        <base />
        <!-- Azure OpenAI BackEnd 연동-->
        <set-backend-service id="apim-generated-policy" backend-id="kms-test-aoai-prd-backend" />
        <!-- # Azure OpenAI Key를 통한 연결 - Azure OpenAI를 API방식으로 통신 -->
        <set-header name="api-key" exists-action="override">
            <value>{{kms-test-aoai-prd-key}}</value>
        </set-header>
        <!-- 'Test-Key' 필수 헤더 키 검증로직 -->
        <choose>
            <!-- <when condition="@( !context.Request.Headers.ContainsKey("TEST-KEY"))"> -->
            <when condition="@( !context.Request.Headers.ContainsKey("TEST-KEY"))">
                <return-response>
                    <set-status code="400" reason="Bad Request (KMS)" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>{"error": "TEST-KEY값이 없음"}</set-body>
                </return-response>
            </when>
            <when condition="@( 
                    context.Request.Headers.ContainsKey("TEST-KEY") && 
                    context.Request.Headers.GetValueOrDefault("TEST-KEY","") != "test"
                    )">
                <return-response>
                    <set-status code="400" reason="Bad Request (KMS)" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>{"error": "TEST-KEY값이 잘못됨"}</set-body>
                </return-response>
            </when>
            <when condition="@( 
                    context.Request.Headers.ContainsKey("TEST-KEY") &&
                    context.Request.Headers.ContainsKey("CS") &&
                    context.Request.Headers.GetValueOrDefault("CS","") == "1"
                    )">
                <return-response>
                    <set-status code="400" reason="Bad Request (KMS)" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>{"error": "Content Safety 위반 1단계"}</set-body>
                </return-response>
            </when>
            <when condition="@( 
                    context.Request.Headers.ContainsKey("TEST-KEY") &&
                    context.Request.Headers.ContainsKey("CS") &&
                    context.Request.Headers.GetValueOrDefault("CS","") == "2"
                    )">
                <return-response>
                    <set-status code="400" reason="Bad Request (KMS)" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>{"error": "Content Safety 위반 2단계"}</set-body>
                </return-response>
            </when>
            <otherwise />
        </choose>
        <!-- 
            PTU관련 접속 제한 설정
            renewal-period : 제한 시간 계산(초단위, 15 : 15초마다 카운터 초기화)
            calls : renewal-period동안 허용되는 최대 호출 횟수(3 : 3번 호출 허용)
            
            <rate-limit-by-key calls="3" renewal-period="15" counter-key="@(context.Subscription.Id)" />
            - calls="3" renewal-period="15" : 각 구독마다 별도로 15초에 3번 호출 허용
            - counter-key : 제한을 적용할 기준 키
                - Subscription.Id : 구독 id를 기준
                - GetValueOrDefault("Ocp-Apim-Subscription-Key","anonymous") : APIM 구독 키를 기준
                    - anonymous : 기본적으로 apim키가 없으면 접근불가(401)
                    - 예외 : Subscription key not required옵션 사용경우 apim키가 없어도 통과
        -->
        <rate-limit-by-key calls="3" renewal-period="15" counter-key="@(context.Request.Headers.GetValueOrDefault("Ocp-Apim-Subscription-Key","anonymous"))" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```
