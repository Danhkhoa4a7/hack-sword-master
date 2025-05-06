
(function () {
    'use strict';

    // -- 1. UI 생성: 좌하단 슬라이더 (1~50000) --
    function createMultiplierUI() {
        // UI 컨테이너
        const container = document.createElement('div');
        container.style.position = 'fixed';
        container.style.bottom = '10px';
        container.style.left = '10px';
        container.style.zIndex = '10000';
        container.style.backgroundColor = 'rgba(0, 0, 0, 0.7)';
        container.style.color = 'white';
        container.style.padding = '8px';
        container.style.borderRadius = '5px';
        container.style.fontFamily = 'Arial, sans-serif';
        container.style.fontSize = '14px';
        container.style.userSelect = 'none';

        // 제목
        const title = document.createElement('div');
        title.innerText = 'Kill Multiplier';
        title.style.marginBottom = '4px';
        container.appendChild(title);

        // 슬라이더
        const slider = document.createElement('input');
        slider.type = 'range';
        slider.min = '1';
        slider.max = '50000';
        slider.value = '1'; // 기본: 1 (즉, 추가 전송 없음)
        slider.style.width = '200px';
        container.appendChild(slider);

        // 값 표시
        const valueLabel = document.createElement('span');
        valueLabel.innerText = ' x1';
        valueLabel.style.marginLeft = '8px';
        container.appendChild(valueLabel);

        slider.addEventListener('input', function () {
            valueLabel.innerText = ' x' + slider.value;
        });

        document.body.appendChild(container);
        return slider;
    }

    // -- 2. WebSocket 메시지 후킹 --
    // 원래 WebSocket.send를 저장
    const originalWSSend = WebSocket.prototype.send;
    // UI에서 kill multiplier 값을 읽기 위해 슬라이더 객체를 생성
    const multiplierSlider = createMultiplierUI();

    WebSocket.prototype.send = function (data) {
        try {
            // data가 문자열이어야 분석 가능
            let messageText;
            if (typeof data === 'string') {
                messageText = data;
            } else if (data instanceof ArrayBuffer || ArrayBuffer.isView(data)) {
                // 문자열로 디코딩 (UTF-8)
                messageText = new TextDecoder("utf-8").decode(data);
            } else {
                messageText = data.toString();
            }

            // 확인: 메시지에 "Client:EnemyController:checkDamage" 가 포함되어 있는지
            if (messageText.indexOf("Client:EnemyController:checkDamage") !== -1) {
                // 기본 메시지 전송 (실제 공격)
                originalWSSend.call(this, data);
                // 추가 전송: 슬라이더 값에 따라 (예: 값이 1이면 추가 전송 없음)
                const multiplier = parseInt(multiplierSlider.value, 10);
                // multiplier의 기본 값 1: 0 extra, 2: 1 extra, ...
                const extraCount = multiplier - 1;
                if (extraCount > 0) {
                    for (let i = 0; i < extraCount; i++) {
                        // 작은 지연 후 추가 전송 (지연 20ms씩)
                        setTimeout(() => {
                            originalWSSend.call(this, data);
                        }, 20 * (i + 1));
                    }
                }
                return; // 이미 처리했으므로 리턴
            }
        } catch (err) {
            console.error("Kill multiplier WS hook error:", err);
        }
        // 일반 메시지는 원래대로 전송
        originalWSSend.call(this, data);
    };

    console.log("[Kill Multiplier] Script loaded and WS.send hooked.");
})();

