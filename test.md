# Longtail 관련 로직 설명

Longtail은 '회계용 결제 리스트'의 결과를 상품별로 집계하여 표현하는 기능.

관련된 테이블이 주문관리자(order-service) 영역이기 때문에, order-service에서 구현됨.

## 소스 구조
* Controller -> AccountingController.java
* Service -> PayAccountingService.java
* QueryDsl -> PayAccountRepositoryImpl.java
* Model -> 
  * Request Model -> AccountingStatisticsReq.java
  * Response Model -> AccountingStatisticsForService.java
  
## 소스 상세
상세 주석 참고

### AccountingController
    * URL : /payment-accounting/statistics/services,
    * Model : AccountingStatisticsReq
~~~java
/**
 * 회계데이터 조회( 기준)
 * 
*/
@GetMapping("/payment-accounting/statistics/services")
public ResponseEntity<List<AccountingStatisticsForService>> getAccountingStatisticsForService(@ModelAttribute AccountingStatisticsReq req) {
    return ResponseEntity.ok(service.getAccountingStatisticsForService(req));
}
~~~
* * *
### PayAccountingService - Longtail 관련 서비스 부분
    ResponseModel 설명
    - (상품+웹이관여부)를 가지고 GroupBy한 결과를 조합하여 보냄
     > part_id : 상품(대분류)
     > last_amount : GroupBy 결과 중 주문과 상품별로 결제 금액 SUM
     > productCount : (Distinct) ORD_NO COUNT
     > team_cd : 결제기본(PAY_PAY_M)의 웹매출이관상태코드(WEB_SALE_TRAN_ST_CD)
        WEB_SALE_TRAN_ST_CD(01) -> pure (순수웹)
        WEB_SALE_TRAN_ST_CD(02) -> transfer (이관웹_
        
~~~java
public List<AccountingStatisticsForService> getAccountingStatisticsForService(final AccountingStatisticsReq req) {
    QPayAccount qPayAccount = QPayAccount.payAccount;
    QOrderProduct qOrderProduct = QOrderProduct.orderProduct;
    QPayment qPayment = QPayment.payment;
    QProductMain qProductMain = QProductMain.productMain;

    // 1) LongTail 관련 데이터 조회
    List<Tuple> ret = repository.getAccountingStatisticsForService(req);

    // 2) 조회된 데이터를 Model로 생성
    List<AccountingStatisticsForService> collect = StreamHelper.asStream(ret).map(result -> {
        if ("advance_package".equals(result.get(qProductMain.productClassCodes.lrgPrdClsCd)) && Objects.requireNonNull(result.get(qProductMain.prdCd)).contains("instantly_package")) {
            // 충전패키지이면서, 즉시등록상품은 실속 S-pack으로 집계되도록 변경 > 2019.07.29 노주희 대리님과 협의
            return AccountingStatisticsForService.builder()
                    .payAmt(result.get(qOrderProduct.payAmt))
                    .asgnTeamCd(result.get(qPayment.webSaleTranStCd))
                    .prdTpCd("saving_package")
                    .productCount(result.get(qPayAccount.ordNo))
                    .prdCd(result.get(qProductMain.prdCd))
                    .build();
        }
        return AccountingStatisticsForService.builder()
                .payAmt(result.get(qOrderProduct.payAmt))
                .asgnTeamCd(result.get(qPayment.webSaleTranStCd))
                .prdTpCd(result.get(qProductMain.productClassCodes.lrgPrdClsCd))
                .productCount(result.get(qPayAccount.ordNo))
                .prdCd(result.get(qProductMain.prdCd))
                .build();
            }
    ).collect(Collectors.toList());

    // 3) Stream GroupBy를 이용해서 (상품, 이관여부)를 가지고 정렬
    return StreamHelper.asStream(collect).collect(Collectors.groupingBy(AccountingStatisticsForService::groupBy)).values().stream()
            .filter(l -> !CollectionUtils.isEmpty(l))
            .map(d -> AccountingStatisticsForService.builder()  // 4) GroupBy된 리스트를 Response Model로 변경
                    .asgnTeamCd(d.get(0).getTeam())
                    .prdTpCd(d.get(0).getPrdTpCd())
                    .productCount(d.stream().map(AccountingStatisticsForService::getProductCount).distinct().count())
                    .payAmt(d.stream().collect(Collectors.groupingBy(AccountingStatisticsForService::detailGroupBy)).values().stream().map(e -> e.get(0).getPayAmt()).reduce(0d, Double::sum))
                    .build()).collect(Collectors.toList());
}
~~~
* * *
### PayAccountRepositoryImpl - Longtail관련 QueryDsl 부분
    관련 테이블 : 
      - PAY_ACNT_USE_PAY_D : 회계용결제정보
      - ORD_ORD_PRD_D : 주문상품정보
      - PAY_PAY_M : 결제기본
      - PRD_PRD_M : 상품기본
~~~java
@Override
public List<Tuple> getAccountingStatisticsForService(AccountingStatisticsReq req) {
    // 1) 필요한 Entity의 QueryDSL 객체 설정
    QPayAccount qPayAccount = QPayAccount.payAccount;
    QOrderProduct qOrderProduct = QOrderProduct.orderProduct;
    QPayment qPayment = QPayment.payment;
    QProductMain qProductMain = QProductMain.productMain;

    // 2) From 절 및 Join 절 설정
    JPQLQuery<PayAccount> from = from(qPayAccount)
            .join(qOrderProduct).on(qPayAccount.ordNo.eq(qOrderProduct.order.ordNo))
            .join(qProductMain).on(qProductMain.prdCd.eq(qOrderProduct.prdCd))
            .join(qPayment).on(qPayAccount.payNo.eq(qPayment.payNo))
            // 3) 기본 조건 절 설정
            .where(qPayment.payMtodTpCd.ne(StringUtils.EMPTY))
            .where(qPayment.payMtodTpCd.ne("admin"))
            .where(qPayment.webSaleTranStCd.ne(StringUtils.EMPTY))
            .where(qPayAccount.selrSaleEmpNo.gt("0"))
            .where(qPayment.mngrDelYn.ne("y"))            .where((qPayment.ocurSrcCd.eq(OccurSource.F1).and(qPayment.mngrPayTpCd.eq("02").and(qPayment.payFshDtm.before(LocalDateTime.of(2019,6,29,0,0))))).not())
            .where(qProductMain.productClassCodes.lrgPrdClsCd.ne("admin_service"))
            ;

    // 4) 동적 조건절 설정
    makeCommonCondition(from, req);

    // 5) SELECT 컬럼 설정
    return from.select(qOrderProduct.payAmt, qPayment.webSaleTranStCd, qProductMain.productClassCodes.lrgPrdClsCd,
            qPayAccount.ordNo, qProductMain.prdCd).fetchResults().getResults();
}
~~~
    동적 조건 절
      - 공통 조건절이기 때문에, Longtail에서 사용하지 않는 조건절도 포함되어 있음
      - 화면상의 모든 조건절이 매칭되는 것은 아님
      - 사용하는 조건 절을 제외한 부분은 주석 처리(설명을 위해)
~~~java
private void makeCommonCondition(JPQLQuery<PayAccount> from, AccountingStatisticsReq req) {
    QPayAccount qPayAccount = QPayAccount.payAccount;
    QPayment qPayment = QPayment.payment;
    // 상품 만료 조건
//        if (req.isValidProduct()) {
//            from.where(byItemIdUsed(req.getValidProduct()));
//        }

    // category 옵션
//        if (req.isCategory()) {
//            from.where(byCategory(req.getCategories()));
//        }

    // 상품명 또는 상품코드
//        if (req.isItem()) {
//            from.where(byItemInfos(req.getPrdCd(), req.getPrdNm()));
//        }

    // 대분류
//        if (req.isLrgPrdClsCd()) {
//            from.where(byLrgPrdClsCd(req.getLrgPrdClsCd()));
//        }

    // 날짜 형식에 따른(신청일, 신청완료일, 결제완료일, 매출발생일)
    if (StringUtils.isNotEmpty(req.getSearchDateType())) {
        bySearchDateType(from, req.getSearchDateType(), req.getStartDt(), req.getEndDt());
    }

//    if (req.isMemNm()) {
//        from.where(qPayAccount.memNm.like(StringUtils.likes(req.getMemNm())));
//    }

//        if (req.isDeptCd()) {
//            if ("web".equals(req.getAsgnDeptCd())) {
//                from.where(qPayment.webSaleTranStCd.ne("01"));
//            }
//
//            if ("sales".equals(req.getAsgnDeptCd())) {
//                from.where(qPayment.webSaleTranStCd.eq("02"));
//            }
//        }

    // 결제 상태 조건
    if (req.isPayResult()) {
        from.where(qPayment.payStCd.eq(req.getPayResult()));
    }


//        if (req.isPayStatus()) {
//            if ("free".equals(req.getPayStatus())) {
//                from.where(qPayAccount.ordPaySchdAmt.lt(1));
//                from.where(qPayAccount.spntUseAmt.lt(1));
//            } else {
//                from.where(qPayAccount.ordPaySchdAmt.gt(0));
//                from.where(qPayAccount.spntUseAmt.gt(0));
//            }
//        }

//        if (req.isUnionType()) {
//            from.where(byUnionType(req.getUnionType()));
//        }

    // 결제구분 조건
    if (req.hasBpayApayTpCd()) {
        from.where(qPayment.bpayApayTpCd.eq(req.getBPayApayTpCd()));
    }

    // 회원구분 조건
    if (req.hasMemTpCd()) {
        from.where(qPayAccount.memTpCd.eq(req.getMemTpCd()));
    }

    // 담당부서 조건
    Optional.ofNullable(req.getAsgnTeamCd()).filter(StringUtils::isNotEmpty).ifPresent(asgnTeamCd -> from.where(qPayAccount.asgnTeamCd.eq(asgnTeamCd)));
    // 담당자 조건
    Optional.ofNullable(req.getSelrSaleEmpNo()).filter(StringUtils::isNotEmpty).ifPresent(selrSaleEmpNo -> from.where(qPayAccount.selrSaleEmpNo.eq(selrSaleEmpNo)));
    // 결제 방법 조건
    Optional.ofNullable(req.getPayMtodTpCd()).filter(StringUtils::isNotEmpty).ifPresent(payMtodTpCd -> from.where(qPayment.payMtodTpCd.eq(payMtodTpCd)));
    // 세금계산서발행유형 조건
    Optional.ofNullable(req.getTxinvIsueYn()).filter(StringUtils::isNotEmpty).ifPresent(txinvIsueYn -> from.where(qPayAccount.txinvIsueYn.eq(txinvIsueYn)));
    // 결제 번호 조건
    Optional.ofNullable(req.getPayNo()).ifPresent(payNo -> from.where(qPayAccount.payNo.eq(payNo)));
    // 주문 번호 조건
    Optional.ofNullable(req.getOrdNo()).ifPresent(ordNo -> from.where(qPayAccount.ordNo.eq(ordNo)));
    // 구매자 조건
    Optional.ofNullable(req.getBuerNm()).filter(StringUtils::isNotEmpty).ifPresent(buerNm -> from.where(qPayment.buerNm.eq(buerNm)));
    // 기업형태 조건
    Optional.ofNullable(req.getComTpCd()).filter(StringUtils::isNotEmpty).ifPresent(comTpCd -> from.where(qPayAccount.comTpCd.eq(comTpCd)));
    // 기업명 조건
    Optional.ofNullable(req.getComNm()).filter(StringUtils::isNotEmpty).ifPresent(comNm -> from.where(qPayAccount.comTpCd.like("%" + comNm + "%")));
    // 주업종 조건
    Optional.ofNullable(req.getBizCnnt()).filter(StringUtils::isNotEmpty).ifPresent(bizCnnt -> from.where(qPayAccount.bizCnnt.eq(bizCnnt)));
    // 회원아이디 조건
    Optional.ofNullable(req.getLoinId()).filter(StringUtils::isNotEmpty).ifPresent(loinId -> from.where(qPayAccount.loinId.eq(loinId)));
}
~~~    
* * *
### 레퍼런스
QueryDsl Reference : http://www.querydsl.com/static/querydsl/4.0.1/reference/ko-KR/html_single/
