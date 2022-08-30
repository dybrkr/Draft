# 문서화 되지 않은 COM 오브젝트 사용방법(QueryInterface)

> Undocumented COM object usage
> 

# Purpose

COM 오브젝트를 다루는 기능을 개발하면서 호환성 문제를 하기위해서 연구를 진행했습니다. 정확히는 공식으로 제공되는 인터페이스에서 다른 문서화 되지 않은 인터페이스를 참조할 수 있는 지 확인했습니다. 결론적으로 비교적 호환성 문제를 덜 발생시킬 수 있는 방안을 찾았습니다. 

## 공식 제공 API

**IUnknown** 는 기본적으로 오브젝트의 레퍼런스를 관리하기 위한 3가지 함수를  포함하고 있습니다.

![UCOU-1](.\image\UCOU-1.png)

이 중 QueryInterface 는 특정 오브젝트에서 참조 가능한 다른 인터페이스를 조회하는 함수입니다. 이를 분석했습니다. 

## **QueryInterface 함수 분석**

일반적으로 Table을 참조하여 인터페이스 조회를 수행합니다. 대략적인 구조는 다음과 같습니다.

```go
type EntryType1 struct { // sizeof() => 0x18
		*UUID GUID // 0x00
		uint64 Offset // 0x08
		uint64 Flag // 0x10 ; always set to 1
}

type EntryType2 struct { // sizeof() => 0x18
		*UUID NULL_GUID // 0x00 ; always set to 0
		uint64 Argument // 0x08
		uint64 Func // 0x10
}
```

만일 인자로 전달된 GUID 가 0 이면 하위 클래스의 메서드를 호출하는 것으로 예상할 수 있습니다.
GUID가 널이 아니라면 객체에서 일정 부분 떨어진 위치의 가상 함수. 테이블의 AddRef 함수를 호출합니다.

이로써 특정 COM 오브젝트에서 조회 가능한 객체를 확인할 수 있습니다.

아래는 분석을 통해 알아낸 정보입니다.

1. QueryInterface 가 참조하는 Table을 확인하면 조회 가능한 객체를 열거 할 수 있다.
2. 특정 객체에서 참조 가능한 다른 객체는 현재 오브젝트 구조 내에 다른 오브젝트 레퍼런스를 가지고 있다.

## 예시

**IDXGIFactoryDWM**

QueryInterface를 이용하여 문서화되지 않은 IDXGIFactoryDWM 인터페이스를 조회할 수 있습니다. 

```
struct __declspec(uuid("{1ddd77aa-9a4a-4cc8-9e55-98c196bafc8f}"))
IDXGIFactoryDWM8 : public IUnknown
{
    virtual HRESULT STDMETHODCALLTYPE CreateSwapChainDWM(
    /* [annotation][in] */ 
    _In_ IUnknown *pDevice, 
    /* [annotation][in] */ 
    _In_ DXGI_SWAP_CHAIN_DESC1 *pSwapChainDesc1, 
    /* [annotation][in] */ 
    _In_ DXGI_SWAP_CHAIN_FULLSCREEN_DESC *pSwapChainFullScreenDesc1, 
    /* [annotation][in] */ 
    _In_ IDXGIOutput * pOutput, 
    /* [annotation][out] */ 
    _Out_ IDXGISwapChainDWM8 ** ppSwapChainDWM1
    ) = 0;

    virtual HRESULT STDMETHODCALLTYPE CreateSwapChainDDA(
    /* [annotation][in] */ 
    _In_ IUnknown *pDevice, 
    /* [annotation][in] */ 
    _In_ DXGI_SWAP_CHAIN_DESC1 *pSwapChainDesc1, 
    /* [annotation][in] */ 
    _In_ IDXGIOutput * pOutput, 
    /* [annotation][out] */ 
    _Out_ IDXGISwapChainDWM8 ** ppSwapChainDWM8
    ) = 0;
}
// https://stackoverflow.com/questions/24739142/how-to-distort-manipulate-dwm-live-thumbnails-in-windows-8-or-in-any-other-way
```

### **DXGIFactoryDWM 객체 생성**

위에서 언급한 것 처럼 DXGIFactory 객체의 QueryInterface를 분석하면 DXGI Factory로 부터 조회 가능한 인터페이스를 확인할 수 있었고 그 결과로 DXGIFactoryDWM 객체를 생성할 수 있음을 확인했습니다.

```
// DXGIFactory 객체
[INFO] dxgiFactory => 00000231CC891DE0

// DXGIFactory::QueryInterface 에서 참조하는 entry table 
0:000> dqs 07ffc`83a00000 + 9CEd0 L?0x100
// ...
00007ffc`83a9cfc0  00007ffc`83aad780 dxgi!GUID_1ddd77aa_9a4a_4cc8_9e55_98c196bafc8f
00007ffc`83a9cfc8  00000000`00000088 // offset 0x88
00007ffc`83a9cfd0  00000000`00000001
// ...

// 오프셋 위치에 존재하는 vftable 포인터
0:000> dqs 00000231cc4d1ff0 +88
00000231`cc4d2078  00007ffc`83a98310 dxgi!ATL::CComObject<CDXGIFactory>::`vftable'

0:000> dqs 00007ffc`83a98310
00007ffc`83a98310  00007ffc`83a28860 dxgi!ATL::CComObject<CDXGIFactory>::QueryInterface
00007ffc`83a98318  00007ffc`83a28160 dxgi!ATL::CComObject<CDXGIFactory>::AddRef
00007ffc`83a98320  00007ffc`83a28a90 dxgi!ATL::CComObject<CDXGIFactory>::Release
00007ffc`83a98328  00007ffc`83a5f2c0 dxgi!CDXGIFactory::CreateSwapChainDWM 
```

![UCOU-2](.\image\UCOU-2.png)

# 결론

특정 오브젝트에서 조회 가능한 인터페이스를 열거할 수 있다.