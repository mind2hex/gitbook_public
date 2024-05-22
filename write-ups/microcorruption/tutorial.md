# Tutorial

{% tabs %}
{% tab title="Assembly" %}
```
0010 <__trap_interrupt>
0010:  3041           ret
4400 <__init_stack>
4400:     [overwritten]
4404 <__low_level_init>
4404:  1542 5c01      mov	&0x015c, r5
4408:  75f3           and.b	#-0x1, r5
440a:  35d0 085a      bis	#0x5a08, r5
440e <__do_copy_data>
440e:  3f40 0000      clr	r15
4412:  0f93           tst	r15
4414:  0724           jz	$+0x10 <__do_clear_bss+0x0>
4416:  8245 5c01      mov	r5, &0x015c
441a:  2f83           decd	r15
441c:  9f4f 8645 0024 mov	0x4586(r15), 0x2400(r15)
4422:  f923           jnz	$-0xc <__do_copy_data+0x8>
4424 <__do_clear_bss>
4424:  3f40 0000      clr	r15
4428:  0f93           tst	r15
442a:  0624           jz	$+0xe <main+0x0>
442c:  8245 5c01      mov	r5, &0x015c
4430:  1f83           dec	r15
4432:  cf43 0024      mov.b	#0x0, 0x2400(r15)
4436:  fa23           jnz	$-0xa <__do_clear_bss+0x8>
4438 <main>
4438:  3150 9cff      add	#0xff9c, sp
443c:  3f40 a844      mov	#0x44a8 "Enter the password to continue", r15
4440:  b012 5845      call	#0x4558 <puts>
4444:  0f41           mov	sp, r15
4446:  b012 7a44      call	#0x447a <get_password>
444a:  0f41           mov	sp, r15
444c:  b012 8444      call	#0x4484 <check_password>
4450:  0f93           tst	r15
4452:  0520           jnz	$+0xc <main+0x26>
4454:  3f40 c744      mov	#0x44c7 "Invalid password; try again.", r15
4458:  b012 5845      call	#0x4558 <puts>
445c:  063c           jmp	$+0xe <main+0x32>
445e:  3f40 e444      mov	#0x44e4 "Access Granted!", r15
4462:  b012 5845      call	#0x4558 <puts>
4466:  b012 9c44      call	#0x449c <unlock_door>
446a:  0f43           clr	r15
446c:  3150 6400      add	#0x64, sp
4470 <__stop_progExec__>
4470:  32d0 f000      bis	#0xf0, sr
4474:  fd3f           jmp	$-0x4 <__stop_progExec__+0x0>
4476 <__ctors_end>
4476:  3040 8445      br	#0x4584 <_unexpected_>
447a <get_password>
447a:  3e40 6400      mov	#0x64, r14
447e:  b012 4845      call	#0x4548 <getsn>
4482:  3041           ret
4484 <check_password>
4484:  6e4f           mov.b	@r15, r14
4486:  1f53           inc	r15
4488:  1c53           inc	r12
448a:  0e93           tst	r14
448c:  fb23           jnz	$-0x8 <check_password+0x0>
448e:  3c90 0900      cmp	#0x9, r12
4492:  0224           jz	$+0x6 <check_password+0x14>
4494:  0f43           clr	r15
4496:  3041           ret
4498:  1f43           mov	#0x1, r15
449a:  3041           ret
449c <unlock_door>
449c:  3012 7f00      push	#0x7f
44a0:  b012 f444      call	#0x44f4 <INT>
44a4:  2153           incd	sp
44a6:  3041           ret
44a8 .strings:
44a8: "Enter the password to continue"
44c7: "Invalid password; try again."
44e4: "Access Granted!"
44f4 <INT>
44f4:  1e41 0200      mov	0x2(sp), r14
44f8:  0212           push	sr
44fa:  0f4e           mov	r14, r15
44fc:  8f10           swpb	r15
44fe:  024f           mov	r15, sr
4500:  32d0 0080      bis	#0x8000, sr
4504:  b012 1000      call	#0x10
4508:  3241           pop	sr
450a:  3041           ret
450c <putchar>
450c:  2183           decd	sp
450e:  0f12           push	r15
4510:  0312           push	#0x0
4512:  814f 0400      mov	r15, 0x4(sp)
4516:  b012 f444      call	#0x44f4 <INT>
451a:  1f41 0400      mov	0x4(sp), r15
451e:  3150 0600      add	#0x6, sp
4522:  3041           ret
4524 <getchar>
4524:  0412           push	r4
4526:  0441           mov	sp, r4
4528:  2453           incd	r4
452a:  2183           decd	sp
452c:  3f40 fcff      mov	#0xfffc, r15
4530:  0f54           add	r4, r15
4532:  0f12           push	r15
4534:  1312           push	#0x1
4536:  b012 f444      call	#0x44f4 <INT>
453a:  5f44 fcff      mov.b	-0x4(r4), r15
453e:  8f11           sxt	r15
4540:  3150 0600      add	#0x6, sp
4544:  3441           pop	r4
4546:  3041           ret
4548 <getsn>
4548:  0e12           push	r14
454a:  0f12           push	r15
454c:  2312           push	#0x2
454e:  b012 f444      call	#0x44f4 <INT>
4552:  3150 0600      add	#0x6, sp
4556:  3041           ret
4558 <puts>
4558:  0b12           push	r11
455a:  0b4f           mov	r15, r11
455c:  073c           jmp	$+0x10 <puts+0x14>
455e:  1b53           inc	r11
4560:  8f11           sxt	r15
4562:  0f12           push	r15
4564:  0312           push	#0x0
4566:  b012 f444      call	#0x44f4 <INT>
456a:  2152           add	#0x4, sp
456c:  6f4b           mov.b	@r11, r15
456e:  4f93           tst.b	r15
4570:  f623           jnz	$-0x12 <puts+0x6>
4572:  3012 0a00      push	#0xa
4576:  0312           push	#0x0
4578:  b012 f444      call	#0x44f4 <INT>
457c:  2152           add	#0x4, sp
457e:  0f43           clr	r15
4580:  3b41           pop	r11
4582:  3041           ret
4584 <_unexpected_>
4584:  0013           reti	pc
```
{% endtab %}

{% tab title="C" %}
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>


void get_password(char *password);
int check_password(const char *s);
int unlock_door();

int main(){

  char password[100];

  while (1){
    
    puts("Enter the password to continue");
    get_password(password);

    if (check_password(password) == 1){
      puts("Access Granted!");
      unlock_door();
    }else{
      puts("Invalid password; try again.");
    }
    
  }

  return 0;
}

void get_password(char *s){
  fgets(s, 100, stdin);
}

int check_password(const char *s){
  // check password length
  int string_length = strlen(s);

  if (string_length  == 9){
    return 1;
  }else{
    return 0;
  }
  
}

int unlock_door(){
  exit(0);
}
```
{% endtab %}
{% endtabs %}

Tutorial is pretty simple, if we focus in `check_password` function, we can see that it only checks for the string length.&#x20;

```
4484 <check_password>
4484:  6e4f           mov.b	@r15, r14
4486:  1f53           inc	r15
4488:  1c53           inc	r12
448a:  0e93           tst	r14
448c:  fb23           jnz	$-0x8 <check_password+0x0>
448e:  3c90 0900      cmp	#0x9, r12
4492:  0224           jz	$+0x6 <check_password+0x14>
4494:  0f43           clr	r15
4496:  3041           ret
4498:  1f43           mov	#0x1, r15
449a:  3041           ret
```

This function counts the length of the password string and if it is equals to `0x9` , then returns 1, otherwise returns 0. The problem here is that this function also counts the `\n` at the end of the string so if we introduce a string with a length of 9 characters, it will fail, so we have to introduce only 8 characters to unlock the door. This way the function will return 1.

```
4438 <main>
4438:  3150 9cff      add	#0xff9c, sp
443c:  3f40 a844      mov	#0x44a8 "Enter the password to continue", r15
4440:  b012 5845      call	#0x4558 <puts>
4444:  0f41           mov	sp, r15
4446:  b012 7a44      call	#0x447a <get_password>
444a:  0f41           mov	sp, r15
444c:  b012 8444      call	#0x4484 <check_password>
4450:  0f93           tst	r15
4452:  0520           jnz	$+0xc <main+0x26>
4454:  3f40 c744      mov	#0x44c7 "Invalid password; try again.", r15
4458:  b012 5845      call	#0x4558 <puts>
445c:  063c           jmp	$+0xe <main+0x32>
445e:  3f40 e444      mov	#0x44e4 "Access Granted!", r15
4462:  b012 5845      call	#0x4558 <puts>
4466:  b012 9c44      call	#0x449c <unlock_door>
446a:  0f43           clr	r15
446c:  3150 6400      add	#0x64, sp
```

After returning from `check_password` function, the result is stored in the register `r15`. The next instruction in `main` is simply a comparasion to test if `r15` is equal to `0x0`, in case `r15` is equals to `0x0`, then no jumps are performed, which result in the printing of the message `"Invalid password...` but if `r15` is equals to one, a jump to `0x445e`will be performed and as result, a call to  `unlock_door` function will be performed also.

```
4450: tst	r15
4452: jnz	$+0xc <main+0x26>
4454: mov	#0x44c7 "Invalid password; try again.", r15
4458: call	#0x4558 <puts>
445c: jmp	$+0xe <main+0x32>
445e: mov	#0x44e4 "Access Granted!", r15
4462: call	#0x4558 <puts>
4466: call	#0x449c <unlock_door>
446a: clr	r15
446c: add	#0x64, sp
```

<figure><img src="../../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>
