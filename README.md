# Team14_BE
14조 백엔드

✏ 9주차 PR 리뷰받고 싶은 내용
---
```java
@Service
@Slf4j
@RequiredArgsConstructor
/** 결제 승인 서비스 */
public class PaymentConfirmService {

	private final TossPaymentsClient tossPaymentsClient;

	private final PaymentValidationService paymentValidationService;
	private final PaymentStatusUpdateService paymentStatusUpdateService;
	private final PointUpdateService pointUpdateService;

	@Transactional
	public PaymentConfirmationResponse confirm(PaymentConfirmRequest request) {
		// 1. 결제 상태 변경 (준비 -> 실행 중)
		paymentStatusUpdateService.updatePaymentStatusToExecuting(request.orderId(), request.paymentKey());
		// 2. 결제 유효성 검사
		paymentValidationService.validate(request.orderId(), BigDecimal.valueOf(request.amount()));
		// 3. 결제 승인 요청
		PaymentConfirmationResponse response = tossPaymentsClient.confirmPayment(request);
		// 4. 승인 결과에 따른 결제 상태 업데이트
		paymentStatusUpdateService.updatePaymentStatus(new PaymentStatusUpdateCommand(request.paymentKey(), request.orderId(), response.paymentStatus()));
		// 5. 포인트 충전
		pointUpdateService.increasePoint(request.orderId());
		return response;
	}

}
```
현재 결제 승인 과정에서 5가지 로직이 위와 같이 한 트랜잭션으로 수행되고 있습니다. <br>
`confirm` 메소드에서 하나의 트랜잭션으로 5개의 로직이 수행될 때 “결제 상태 변경”과 같은 부가기능의 실패로 결제 승인에 성공하였음에도 <br> 
DB에 올바르게 기록되지 못하는 상황을 방지하기 위해 트랜잭션을 분리할 수 있나요?

현재 결제 승인 내부 로직 전체를 한 트랜잭션으로 수행 중인데 여기서 수행되는 몇몇의 기능은 별도의 트랜잭션에서 수행되도록 구현하고 싶은데 <br>
어떤 방법이 좋을지 궁금합니다!
