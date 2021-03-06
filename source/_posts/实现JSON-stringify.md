---
title: 实现JSON.stringify
date: 2020-04-30 21:20:20
---

![code.png](https://i.loli.net/2020/05/01/jts1yGrNJ73fRaD.png)

<!-- more -->

测试代码如下：

```typescript
import 'mocha';
import { expect } from 'chai';
import { stringify } from '../../src/json';

describe('JSON stringify', () => {
  it('should parse string', () => {
    expect(stringify('string')).to.equal('string');
  });
  it('should parse number', () => {
    expect(stringify(1)).to.equal('1');
  });
  it('should parse null', () => {
    expect(stringify(null)).to.equal('null');
  });
  it('should parse undefined', () => {
    expect(stringify(undefined)).to.equal('undefined');
  });
  it('should parse empty array', () => {
    expect(stringify([])).to.equal('[]');
  });
  it('should parse flat array', () => {
    expect(stringify([1, 2, 3])).to.equal('[1, 2, 3]');
  });
  it('should parse nest array', () => {
    expect(stringify([1, 2, { a: [3, 4, 5] }])).to.equal(
      '[1, 2, {"a": [3, 4, 5]}]',
    );
  });
  it('should parse empty object', () => {
    expect(stringify({})).to.equal('{}');
  });
  it('should parse flat object', () => {
    expect(stringify({ a: 1 })).to.equal('{"a": 1}');
  });
  it('should parse nest object', () => {
    expect(stringify({ a: 1, b: 2, c: { d: 1 } })).to.equal(
      '{"a": 1, "b": 2, "c": {"d": 1}}',
    );
  });
  it('should resolve circular dependency', () => {
    const a: any = {};
    const b: any = {};
    b.a = a;
    a.b = b;
    expect(() => {
      stringify(a);
    }).to.throw('Circular Dependency');
  });
  it('should resolve same reference', () => {
    const a: any = {};
    const o: any = { d: 1 };
    a.b = o;
    a.c = o;
    expect(stringify(a)).to.equal('{"b": {"d": 1}, "c": {"d": 1}}');
  });
});
```
