// Copyright 2015 go-smpp authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

package smpp

import (
	"crypto/tls"
	"fmt"
	"math/rand"
	"time"

	"gitlab.services.mts.ru/web-push-service/go-smpp/smpp/pdu"
	"gitlab.services.mts.ru/web-push-service/go-smpp/smpp/pdu/pdufield"
)

// Transceiver implements an SMPP transceiver.
//
// The API is a combination of the Transmitter and Receiver.
type Transceiver struct {
	Addr            string
	User            string
	Passwd          string
	SystemType      string
	EnquireLink     time.Duration
	RespTimeout     time.Duration
	TLS             *tls.Config
	WindowSize      uint
	Handler         HandlerFunc
	ConnInterceptor ConnMiddleware

	Transmitter
}

//
// f is only set on transceiver.
func (t *Transceiver) handlePDU() {
	for {
		p, err := t.conn.Read()
		if err != nil {
			break
		}
		switch p.Header().ID {
		case pdu.DeliverSMID:
			pResp := pdu.NewDeliverSMRespSeq(p.Header().Seq)
			t.conn.Write(pResp)
		case pdu.DataSMID:
			pResp := pdu.NewDataSMRespSeq(p.Header().Seq)
			t.conn.Write(pResp)
		}
		t.Handler(p)
	}
}

// Bind implements the ClientConn interface.
func (t *Transceiver) Bind() <-chan ConnStatus {
	t.r = rand.New(rand.NewSource(time.Now().UnixNano()))
	t.conn.Lock()
	defer t.conn.Unlock()
	if t.conn.client != nil {
		return t.conn.Status
	}
	t.tx.Lock()
	t.tx.inflight = make(map[uint32]chan *tx)
	t.tx.Unlock()
	c := &client{
		Addr:            t.Addr,
		TLS:             t.TLS,
		EnquireLink:     t.EnquireLink,
		RespTimeout:     t.RespTimeout,
		Status:          make(chan ConnStatus, 1),
		BindFunc:        t.bindFunc,
		WindowSize:      t.WindowSize,
		ConnInterceptor: t.ConnInterceptor,
	}
	t.conn.client = c
	c.init()
	go c.Bind()
	return c.Status
}

func (t *Transceiver) bindFunc(c Conn) error {
	p := pdu.NewBindTransceiver()
	f := p.Fields()
	f.Set(pdufield.SystemID, t.User)
	f.Set(pdufield.Password, t.Passwd)
	f.Set(pdufield.SystemType, t.SystemType)
	resp, err := bind(c, p)
	if err != nil {
		return err
	}
	if resp.Header().ID != pdu.BindTransceiverRespID {
		return fmt.Errorf("unexpected response for BindTransceiver: %s",
			resp.Header().ID)
	}
	go t.handlePDU()
	return nil
}
