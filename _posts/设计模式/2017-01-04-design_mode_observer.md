---
layout: post
title: 观察者模式
categories: 设计模式
date: 2017-01-04 19:10:11 -0800
pid: 20170104-123811
---
> 观察者模式也叫发布订阅模式

``` ruby
package main

import "container/list"

type Subject interface {
	Attach(Observers)
	Detach(Observers)
	Notify()
}

type Observers interface {
	Update(Subject)
}

type NbaSubject struct {
	observers *list.List
	value     int
}

func NewNbaSubject() *NbaSubject {
	s := new(NbaSubject)
	s.observers = list.New()
	return s
}

func (s *NbaSubject) Attach(observer Observers) {
	s.observers.PushBack(observer)
}

func (s *NbaSubject) Detach(observer Observers) { //释放观察者
	for ob := s.observers.Front(); ob != nil; ob = ob.Next() {
		if ob.Value.(*Observers) == &observer {
			s.observers.Remove(ob)
			break
		}
	}
}

func (s *NbaSubject) Notify() {
	for ob := s.observers.Front(); ob != nil; ob = ob.Next() {
		ob.Value.(Observers).Update(s)
	}
}

func (s *NbaSubject) setValue(value int) {
	s.value = value
	s.Notify()
}

func (s *NbaSubject) getValue() int {
	return s.value
}

type NbaObserver1 struct {
}

func (c *NbaObserver1) Update(subject Subject) {
	println("NbaObserver1  value is ", subject.(*NbaSubject).getValue())
}

type NbaObserver2 struct {
}

func (c *NbaObserver2) Update(subject Subject) {
	println("NbaObserver2 value is ", subject.(*NbaSubject).getValue())
}

func main() {
	subject := NewNbaSubject()
	ob1 := new(NbaObserver1)
	ob2 := new(NbaObserver2)
	subject.Attach(ob1)
	subject.Attach(ob2)
	subject.setValue(0)
	subject.setValue(5)
}
```
