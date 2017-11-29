### Wed Nov 29 2017 14:15

The C# compiler implements closures as compiler generated sealed classes with un-typeable names like `'<>c__DisplayClass0_0'`. These classes can maintain state in instance variables, and provide an invoke method (with a similarly un-typeable name like `<Test>b__0`) which is used to instantiate `Func`s

```cs
// source
using System;
public class Program {
    Func<int, float> Test(float a) {
        return (x) => x * a;
    }

    void UseTest(int b) {
        var x = Test(99);
        x(b);
    }
}
```

```cs
// disassembly
public class Program
{
    [CompilerGenerated]
    private sealed class <>c__DisplayClass0_0
    {
        public float a;

        internal float <Test>b__0(int x)
        {
            return (float)x * this.a;
        }
    }

    private Func<int, float> Test(float a)
    {
        Program.<>c__DisplayClass0_0 expr_05 = new Program.<>c__DisplayClass0_0();
        expr_05.a = a;
        return new Func<int, float>(expr_05.<Test>b__0);
    }

    private void UseTest(int b)
    {
        this.Test(99f)(b);
    }
}
```

Conversion to a `Func` happens with the `ldftn` opcode in the `Test` method.

```msil
.method private hidebysig
    instance class [mscorlib]System.Func`2<int32, float32> Test (
        float32 a
    ) cil managed
{
    // Method begins at RVA 0x2050
    // Code size 24 (0x18)
    .maxstack 8

    IL_0000: newobj instance void Program/'<>c__DisplayClass0_0'::.ctor()
    IL_0005: dup
    IL_0006: ldarg.1
    IL_0007: stfld float32 Program/'<>c__DisplayClass0_0'::a
    IL_000c: ldftn instance float32 Program/'<>c__DisplayClass0_0'::'<Test>b__0'(int32)
    IL_0012: newobj instance void class [mscorlib]System.Func`2<int32, float32>::.ctor(object, native int)
    IL_0017: ret
} // end of method Program::Test
```

This can be reproduced by hand in C#. We can even tweak the semantics to make the fields closed over immutable, more in line with a functional language.

```cs
using System;
public class Program {
    sealed class Closure {
        readonly float a;
        internal Closure(float a) { this.a = a; }
        internal float __Invoke(int x) { return x * a; }
    }

    Func<int, float> Test(float a) {
        return new Func<int,float>(new Closure(a).__Invoke);
    }

    void UseTest(int b) {
        var x = Test(99);
        x(b);
    }
}
```

The disassembly is largely identical to the source in this case, but the `Func` construction is a little different.

```msil
.method private hidebysig
    instance class [mscorlib]System.Func`2<int32, float32> Test (
        float32 a
    ) cil managed
{
    // Method begins at RVA 0x2050
    // Code size 18 (0x12)
    .maxstack 8

    IL_0000: ldarg.1
    IL_0001: newobj instance void Program/Closure::.ctor(float32)
    IL_0006: ldftn instance float32 Program/Closure::__Invoke(int32)
    IL_000c: newobj instance void class [mscorlib]System.Func`2<int32, float32>::.ctor(object, native int)
    IL_0011: ret
} // end of method Program::Test
```
