#ABAP #ALV #In_Progress #Technical

# 1. Table / Structure의 필드에 대해서 Field Catalog Setting을 관리하는 프로그램. 
- 프로그램 이름 : ZPRG_FIELDCAT_SETTING
- 개발 의도
	- 기존에는 Grid ALV를 사용하는 프로그램이 매번 field catalog를 세팅해줘야 하는데. 
	- Field Catalog를 안 쓰는 SALV의 경우에는 customizing이 안되는 단점이 있어서 Grid ALV를 어쩔 수 없이 사용하는 경우
	- Field catalog 설정을 별도의 관리 테이블에 저장하여 프로그램별 출력 제어를 가능하게 한다. 
	- 프로그램 별로 출력하는 테이블이 다른 경우에, 리포트 별로 field catalog를 관리할 것. 
- 관리 대상
	- ==**ALV 출력 라인의 구조를 정의하는 DDIC Object**==를 원칙으로 한다.
		- Transparent Table도 내부적으로 STRUCTURE로 간주한다. 
		- Join 또는 계산 결과를 정의한 DDIC Structure. 
- CBO TABLE인 ==**ZTAB_FIELDCAT_SETTING**==에서 저장 및 조회

## 1-1. CBO TABLE의 필드
### 1-1-1. 테이블 설정. 
- 테이블 이름 : ZTAB_FIELDCAT_SETTING
- KEY FIELD
	- Program Name : Field catalog 설정을 사용하는 리포트 프로그램 명. 
	- ALV ID : 동일 프로그램 내 여러 ALV를 사용하는 경우의 ALV 구분자. 
	- Table / DDIC Structure Name : ALV 출력 라인의 구조를 정의하는 Table 또는 Structure. 
	- Field Name : 개별 필드. 
- Non-key Fields
	- Field catalog 관련 설정 항목 (LVC_S_FCAT의 총 107개 항목 중)
		- no_out
		- ALV_editable(alv 출력시 사용자가 수정할 수 있느냐)
		- output length
		- COLTEXT
		- col_pos
		- just
		- HOTSPOT
		- EMPHASIZE
	- 상태, 이력 필드. 
		- changed_by / time / date
		- created_by / time / date
		- ACTIVE flag

## 1-2. 고려 사항.
### 1-2-1. Key 정책
- 해당 프로그램에서 관리하는 ZTAB_FIELDCAT_SETTING의 key field에 대해서. 
	- PROGRAM NAME
	- ALV ID
	- TABLE / DDIC STRUCTURE NAME
	- FIELD NAME. 
- ALV ID는 동일 프로그램, 동일 테이블 혹은 Structure를 사용하지만 다르게 표시해야 하는 alv를 위한 것임. 
- 프로그램에서 ==JOIN을 통해서 여러 테이블의 필드를 출력==하는 경우, 반드시 ==**출력 단계의 필드를 정의한 DDIC Structure를 생성**==해야 함. 
	- DDIC STRUCTURE : ALV 출력라인의 구조를 정의하는 DDIC OBJECT. 
	- 실제 테이블만을 쓰는 경우와
	- JOIN / 계산 결과에 대해서 정의한 Structure를 포괄. 

### 1-2-2. DDIC 변경 SYNC
- Active Flag, No_out에 대한 정의. 
	- Active Flag
		- Field Catalog 설정 record의 유효 여부를 나타낸다. 
		- ALV 출력 프로그램에서는 관리 테이블에서 ACTIVE FLAG = 'X'인 RECORD만 조회할 것. 
	- No_out
		- active flag와 별개로, 해당 column을 ALV 화면에 표시할지의 여부
- 어떤 모종의 이유로 Table, DDIC Structure의 구조가 바뀌는 경우가 있다.
	- 1차적으로 기존에 저장된 세팅 사항(ZTAB_FIELDCAT_SETTING)을 불러올 것. 
	- 그리고 DDIC 구조를 불러와서
		- 저장된 것 이외의 새로운 필드가 추가되었다면 
			- 새로운 행을 추가해서 모든 필드가 나열되도록 만들고.
			- ACTIVE = 'X'.
			- no_out = ''.
		- 저장된 내역과 비교해서 삭제된 필드가 있다면
			- row color를 다르게 출력(붉은 배경)하고
			- 자동으로 ACTIVE flag = ''를 통해서 비활성 처리. 
	- 최종적으로 사용자가 저장 버튼을 누르면 그 내용대로 CBO TABLE에 저장되고, alv를 출력하는 프로그램에서는 active flag = 'X'인 건만을 기준으로 field catalog를 구성한다. 
- sync 시 기존 필드의 설정값은 유지하며, 신규 필드만 기본값으로 추가한다. 
	- 기본적으로 화면에 표시되는 것을 기본값으로(no_out = '')하고, 관리자는 저장 전에 표시 여부를 조정할 수 있다. 

### 1-2-3. 해당 프로그램에서 Edit 기능의 필요성 여부. 
- EDIT 기능에 대한 정의
	- ZPRG_FIELDCAT_SETTING 프로그램 내에서 관리하는 필드의 값 수정 여부에 대한 내용임. 
	- ==**Field Catalog 설정 중 UI에 대한 EDIT을 의미하는 것이 아님**==. 
- 기본적으로 DDIC에서 바로 구조를 받아오기 때문에, 행 추가의 필요성이 없다. 
- 기본적으로 모든 필드에서 edit이 가능하지만 이력관리용 필드는 non-editable. 
	- changed_by/time/date
	- created_by/time/date.
	- active_flag.
### 1-2-4. Override 규칙
- ALV를 출력하는 프로그램 입장에서는, ZTAB_FIELDCAT_SETTING 테이블에서 정의한 필드의 설정을 우선적으로 반영하게 되지만. 
- 만약 기본값, initial value인 경우에는 DDIC 기본 field catalog를 가능하도록 설정 (이 내용은 ALV를 출력하는 프로그램에서 처리해야 할 내용)
- 값이 ==유효하게== 입력된 경우에만 기본 field catalog를 override하고 입력되도록 한다. 
	- 숫자 필드는 0 또는 Initial인 경우에는 '미입력'으로 간주하고, baseline을 유지하도록 한다. 

### 1-2-5. Validation / Preview / Reset 기능 (추후 개발)
- [ ] menu bar 버튼으로 reset 기능. 
	- Reset 기능은 DDIC 기본 field catalog 상태로 되돌리는 것을 의미함. 
- [ ] validation
	- [ ] col_pos가 중복인지. 
	- [ ] just(정렬 방향, 왼쪽/가운데/오른쪽) 유효성(F4?)
- [ ] preview 기능(추후 결정)
	- [ ] 현 fieldcatalog 설정을 적용한 빈 alv를 출력하기. 
	- [ ] 실제 데이터의 조회 없이 현재 FIELD CATALOG 설정이 적용된 빈 ALV 레이아웃을 확인하기 위한 목적으로 한다. 



# 1-3. 세부 개발방향
- Program Title
	- ZPRG_FIELDCAT_SETTING
	- ALV 필드 카탈로그 관리 프로그램. 
- Selection Screen. 
	- Parameter로 프로그램 이름, 사용하는 테이블을 입력할 것. 
		- program name
		- alv id
		- structure name(table이 될 수도 있고, ddic structure가 될 수도 있고)
			- 입력시에는 table, structure 모두 허용하지만, 내부 처리 시에는 항상 DDIC 구조로 취급한다. 
				- 테이블이면 LINE 구조로 변환 및 해석한다. 
			- DDIC OBJECT가 반드시 존재해야 함. 없으면 에러 메시지 출력 후 SELECTION SCREEN으로 RETURN.
	- Radio button으로 모드를 구분할 수 있도록. 
		- 기본적으로 ddic object 값에 대한 유효성 검증 실패 시 selection screen으로 return한다. 
		- 생성 모드
			- 입력 key 조합이 ZTAB_FIELDCAT_SETTING에 없어야 한다. 
			- 입력한 KEY 조합에 따라서 DDIC STRUCTURE의 필드를 읽어오고, 자동으로 ROW를 생성한다. 
			- 저장하면 ZTAB_FIELDCAT_SETTING에 저장할 수 있도록. 
		- 조회 및 수정 모드. 
			- 입력 KEY 조합이 ZTAB_FIELDCAT_SETTING에 존재해야 한다. 
			- 진입시 자동으로 SYNC를 수행할 것. 
			- 그러나 테이블 데이터에 반영하는 것은 사용자가 **저장** 버튼을 누른 후에야 수행한다. 
				- 저장 버튼 -> 팝업(Y/N) -> COMMIT.
	- selection screen block 하단에 comment를 추가할 수 있도록. 
		- 프로그램에 대한 설명. 
- ALV 화면.
	- GRID ALV로 editable하게 조절함. 
	- 입력한 테이블 이름 또는 Structure를 통해서 어떤 필드가 존재하는지를 파악하고, 자동으로 그에 맞는 row가 출력됨. 
		- internal table에서는 기본적으로 program name, alv id, table name, field name이 기본으로 입력되어 있고, 해당 필드는 key임. 
			- ALV ID는 동일 프로그램 내에서 여러 alv를 출력하는 경우에, 이를 구분하기 위함임. 
		- field catalog에 대한 필드들이 추가로 붙어있음. 
	- 생성 모드 (KEY 조합이 존재하지 않음)
		- DDIC STRUCTURE 필드 목록을 읽어서
			- 만약 KEY 조합이 존재하는 경우 에러 메세지를 출력하고 다시 SELECTION SCREEN으로 RETURN. 
		- 모든 필드에 대하여 ROW를 초기값으로 생성(화면 버퍼)
			- 추가된 필드는 기본적으로 ACTIVE = 'X' (활성화), NO_OUT = ''.
			- col_pos는 비워두고 baseline인 DDIC 순서를 따른다. 
		- 저장 버튼을 누르면 한번에 ZTAB_FIELDCAT_SETTING에 INSERT를 실시. 
	- 조회 및 수정 모드 (KEY 조합이 존재함)
		- DDIC STRUCTURE 필드 목록과 동시에 테이블 데이터를 읽고, 
			- 기존에 있는 내용에 대해서는 저장 버튼을 누르면 update를 한다. 
		- DDIC STRUCTURE를 기준으로 필드 목록이 일치하지 않으면. 
			- 없는 필드는 ACTIVE = '' (비활성화), 해당 ROW를 붉은 배경으로. 
				- 사용자 실수를 제한하기 위해서 필드 목록에 없는 RECORD는 ACTIVE - ''인 상태로 고정, 물리 삭제는 하지 않는다. 
				- 편집 자체를 막도록 non-editable 처리를 한다. (이력필드 non-editable과 같은 맥락)
			- 추가된 필드는 새로 보여주고 초록 배경으로. 
				- 화면 버퍼에 반영됨. 
				- 저장 시에 ZTAB_FIELDCAT_SETTING에 INSERT됨. 
	- ALV MENU BUTTON을 통해서
		- default = display, 버튼 누르면 edit으로 바뀌도록 할 것임. 
		- Save 버튼이 있음. 
	- field catalog 관련 필드
		- 기본 항목. 
			- no_out
			- ALV_editable(alv 출력시 사용자가 수정할 수 있느냐)
			- output length
			- COLTEXT
			- col_pos
			- just
			- HOTSPOT
			- EMPHASIZE
		- 세부 항목. 
			- currency, quantity
				- 초기 버전에서는 출력 적용 대상에서 제외(추후 확장)
			- 그외 추가?
	- 관련되지 않은 필드(상태 필드)
		- created_by
		- created_time
		- created_date
		- changed_by
		- changed_time
		- changed_date.
		- ACTIVE flag

---

# 2. ALV 출력 프로그램에서의 개념 순서도 

## 0) 전제

- 출력 프로그램은 **CL_GUI_ALV_GRID**를 사용한다.
- 출력 데이터는 **DDIC Structure(출력 라인 구조)** 를 기준으로 구성한다.
- Field Catalog 커스터마이징은 **ZTAB_FIELDCAT_SETTING**을 통해 관리한다.

## 1) 출력 데이터 준비

- 리포트 로직(SELECT/JOIN/계산)을 통해 데이터를 조회/가공한다.
- 최종 결과는 **출력용 DDIC Structure 타입의 internal table**(예: `GT_OUT`)에 담는다.  
    (JOIN/계산 결과는 반드시 출력용 DDIC Structure로 정의된 필드에 매핑한다.)


## 2) 출력 라인 구조 확정

- `GT_OUT`의 line type에 해당하는 **출력용 DDIC Structure 명**을 확정한다.
- 해당 Structure 명은 Field Catalog 설정 조회 시 Key의 일부로 사용한다.  
    (Key: Program Name + ALV ID + DDIC Structure Name + Field Name)

## 3) DDIC 기반 baseline Field Catalog 생성 (최초 1회)

- Grid에 데이터를 바인딩하기 직전(최초 표시 시점)에,  
    출력용 DDIC Structure를 기반으로 **baseline Field Catalog**를 생성한다.
- baseline Field Catalog는 DDIC로부터 자동으로 채워지는 항목을 포함한다.
    - 데이터 타입/길이/소수점
    - conversion exit
    - 통화/수량 참조
    - 기본 텍스트(DDIC 기반)

> 목적: ALV의 “자동 추론” 영역을 명시적으로 생성하여, 이후 커스터마이징 merge가 가능하도록 한다.

## 4) 설정 테이블 조회 (ACTIVE 필터 적용)

- `ZTAB_FIELDCAT_SETTING`을 다음 조건으로 조회한다.
    - Program Name
    - ALV ID
    - DDIC Structure Name
    - **ACTIVE FLAG = 'X'**
- 조회 결과는 “유효한 설정 record”로 간주한다.  
    (ACTIVE는 “설정 record의 유효성”, NO_OUT은 “표시 여부”로 서로 역할이 다름)

### 5) baseline + 설정값 merge (Override 규칙 적용)

- baseline Field Catalog의 `FIELDNAME` 기준으로 설정 record를 매칭한다.
- 각 속성에 대해 아래 규칙으로 최종 Field Catalog를 만든다.
    - 설정값이 **initial**이면 → **baseline 값 유지**
    - 설정값이 **유효하게 입력된 경우에만** → 해당 속성을 **override**
	    - 숫자 필드는 0, INITIAL인 경우에는 미입력으로 간주한다. 
	    - 플래그 성격의 필드는 공백값도 유효한 설정값으로 간주한다. 
		    - no_out, hotspot, emphasize, alv_editable 등.
- NO_OUT(표시 여부), COL_POS(순서), 텍스트, JUST, ALV_EDITABLE(출력 수정 가능) 등은 이 단계에서 반영된다.

### 6) 최종 Field Catalog 정렬/정리

- `COL_POS`가 존재하는 경우, 해당 값 기준으로 컬럼 순서를 정렬한다.
- 일부 필드만 `COL_POS`가 있는 경우:
    - `COL_POS`가 있는 필드를 우선 정렬하고,
    - 나머지는 baseline(DDIC) 순서를 유지한다.
- (선택) 최소 검증:
    - DDIC 구조에 존재하지 않는 FIELDNAME은 무시 또는 경고 처리
    - COL_POS 중복/비정상 값은 해당 필드를 col_pos 미입력으로 간주하고 baseline 순서를 따른다. 

### 7) Grid 최초 바인딩 및 출력

- 최종 Field Catalog와 출력 internal table(`GT_OUT`)을 사용하여  
    `CL_GUI_ALV_GRID`의 최초 표시 메서드(예: set_table_for_first_display)에 전달한다.
- 이후 단순 refresh가 필요한 경우에는 Field Catalog 재생성 없이 refresh로 갱신한다.  
    (출력 구조가 고정이므로 baseline 생성은 일반적으로 최초 1회로 충분)

---

## 🔎 보충 정의 (문서 내 재사용용)

- **ACTIVE FLAG**: 설정 record의 유효 여부. 출력 프로그램은 ACTIVE='X'인 record만 조회 대상으로 함.
- **NO_OUT**: 유효한 필드 중 ALV 화면 표시 여부를 제어하는 UI 속성.
- **관리 프로그램의 Edit**: 설정 테이블 화면에서의 수정 가능 여부(UI 제어)이며, ALV 출력 컬럼의 EDIT 속성과는 별개.


---
