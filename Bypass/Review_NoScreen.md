# Review: NoScreen

> https://github.com/dybrkr/NoScreen
forked from https://github.com/KANKOSHEV/NoScreen
> 

### 구현 방식

GetDisplayAffinity 을 통해 Affinity 설정 값을 체크하는 것을 우회하는 방식으로 구현되어있음

커널에서 win32kfull::ChangeWindowTreeProtection 를 직접 호출하여 Affinity와 관련된 Atom 값을 설정하지 않도록하여 우회하고 있습니다.

### A**nalysis**

**GetDisplayAffinity**

WinAPI 함수 GetWindowDisplayAffinity를 호출하면 커널에서는 GetDisplayAffinity가 호출 됩니다. 해당 함수는 해당 윈도우의 Affinity atom값을 반환합니다.

```
// NtUserGetWindowDisplayAffinity -> GetDisplayAffinity
bool GetDisplayAffinity(struct tagWND * wnd, DWORD *affinity)
{
  *affinity= 0;
  
	if(IsTopLevelWindow(wnd) && TestWF(wnd,WEFLAYERED)){
		*affinity= GetProp(wnd, atomDispAffinity, 1);
		result = true;
	}

	return false;
 }
```

**SetDisplayAffinity**

만약 ChangeWindowTreeProtection를 직접 호출하면 atom값이 설정되는 부분이 생략되게 됩니다.

해당 함수는 GetDisplayAffinity와는 반대로 윈도우의 Affinity 설정을 변경하는 함수입니다.  아래처럼 설정할 Affinity flag가 WDA_EXCLUDEFROMCAPTURE 인 경우 ChangeWindowTreeProtection함수를 호출하고 실패 시 원복을 합니다.  

```cpp

bool SetDisplayAffinity(struct tagWND *wnd, DWORD affinity)
{
	if(TestWF(wnd,WEFPREDIRECTED))
		ComposeWindowIfNeeded(wnd);
	
	DWORD old_affinity
	if(GetDisplayAffinity(wnd, &old_affinity))
	{
		if(affinity)
		{
			if(!InternalSetProp(wnd, atomDispAffinity, affinity, 5))
        return false;
    }else{
			// remove
		}
	}
	
bool result = false;
	if (old_affinity & WDA_EXCLUDEFROMCAPTURE != affinity & WDA_EXCLUDEFROMCAPTURE)
	{
		bool result = ChangeWindowTreeProtection(wnd, affinity & WDA_EXCLUDEFROMCAPTURE);
      if(!result)
        InternalSetProp(wnd, atomDispAffinity, affinity , 5);
	}
	return result;
}
```

SetDisplayAffinity 함수에서 ChangeWindowTreeProtection 의 성공 여부에 따라 atomDispAffinity 값을 설정하기 때문에 ChangeWindowTreeProtection 함수를 직접 호출하게 된다면 atom 값을 설정하지 않고 보호할 수 있게 됩니다.



**ChangeWindowTreeProtection**

이제 우회 방식에서 핵심 함수인 ChangeWindowTreeProtection를 보면 자신과 자신과 관련된 윈도우 리스트를 구하고 해당 윈도우의 비트맵을 보호하는 것을 확인할 수 있습니다.

```cpp
bool ChangeWindowTreeProtection(struct tagWND *wnd, unsigned int flag){
	auto wnd_list = BuildHwndList(wnd);
	if(!wnd_list)
	{
		return false;
	}
	
	bool result = flase
	DynamicArray<tagWND*> *array = null_ptr; 
	int size = 0;

  // not accuratly; 
	if(DynamicArray_Add(&array, &size, &wnd) >= 0){
		auto root_ppi = null_ptr; 
		auto pti = wnd->pti;

		if(pti->root == wnd)
		{
			if(pti->p)
				root_ppi = pti-> root??-> ppi
		}	
		for (auto iter = array.begin(); iter != array.end(); iter++)
		{
			auto i_wnd = HMValidateHandleNoSecure(*iter, 1)
	
			if( i_wnd && TestWF(i_wnd ,WEFPREDIRECTED))
			{
				if (flag & 0x1){
					if(i_wnd->ppi != pti->ppi && root_ppi->pi != i_wnd->ppi )
						goto CleanUp
				}

				if (DynamicArray_Add(&array, &size, &i_wnd) < 0)
					goto CleanUp
			}
		}
	}
	
	result = true;

	if(size)
	{
		// loop 
		if(!ProtectWindowBitmap(array[i], flag))
		// ... 
	}
CleanUp:
	FreeHwndList(wnd_list);
	if(array)
		Win32FreePool(array);
}
```

ProtectWindowBitmap 함수를 호출하여 윈도우의 비트맵을 보호합니다. 동작 방식은 flag가 설정되어있을 시에 해당 윈도우 비트맵 오너를 핸들 프로세스로 변경하고 GreProtectSpriteContent 함수를 호출하여 보호합니다.

```cpp
bool ProtectWindowBitmap(struct tagWND *wnd, DWORD flag)
{
  if ( flag & 1 ) 
    pid = wnd->pti->ppi->pid
  else
    pid = 0;

  bool res = ChangeRedirectionBitmapOwner(wnd, pid);
  if ( res )
  {
    if (TestWF(wnd,WEFLAYERED))
    {
      isWDComposed = IsWindowDesktopComposed(wnd);
      res = GreProtectSpriteContent(wnd, *wnd, isWDComposed, flag);

      if(!res)
      {
        if ( flag & 1 )
          ChangeRedirectionBitmapOwner(wnd, 0);
      }
    }
  }
  return res;
}
```

GreProtectSpriteContent 함수에서는 DWMSPRITEREF 구조체 0xa4 위치에 flag 값을 적용하는 것을 볼 수 있습니다.

```cpp
bool GreProtectSpriteContent(struct tagWND *wnd, HWND hwnd, isWDComposed, flag){
	// 생략
	DWMSPRITEREF *ref;
	DWMSPRITEREF::DWMSPRITEREF((DWMSPRITEREF *)ref, hwnd,0,0);
	if (ref)
	{
		old = *(ref + 0xA4)

		is0x11 = (flag & 0x11) == 0x11;
		is0x01 = flag & 0x1

		// mov     [rdi+0A4h], eax
		*(ref + 0xA4) = (v7 << 6) | (old & 0xFFFFFFF7 | (8 * is0x01 )) & 0xFFFFFFBF;
	}
}
```

### 탐지 방법론(내 의견)

1. 리다이렉트 비트맵 오너 변경 확인?

    ```
    // user32!GetPropW
    LOWORD(v3) = GlobalFindAtomW(v3);
        if ( (_WORD)v3 )
          return (HANDLE)NtUserGetProp(v4, (unsigned __int16)v3);
    
    result = (HWND)GetPropW(v1, (LPCWSTR)0xA920);
      else
        result = 0i64;
      return result;
    }
    
    // win32u!NtUserGetProp
    // win32kfull!NtUserGetProp
    
    릴라이어블 하게 가능 ?
    ```

2. tagWND 필드 확인?

    릴라이어블 하게 가능할지.. ?
