---
layout: post
title: "Comparison of distributed Skype alternatives"
description: ""
category: foss
tags: [foss, communication]
---

In this post I'll attempt to perform a comparison between different distributed
skype alternatives that are available.

I will only be looking into FOSS, which eliminates projects like Bittorrent
Bleep.
Transparency is one of the key aspects of building a secure platform.

So the ones that qualify are [Tox](https://tox.chat), and
[Ring](https://ring.cx).

<!-- more -->

## Side-by-Side Facts

<table class="comparison">
  <tr>
    <th></th>
    <th width="50%">Tox</th>
    <th width="50%">Ring</th>
  </tr>
  <tr>
    <td class="row-title">Language:</td>
    <td>C</td>
    <td>C++</td>
  </tr>
  <tr>
    <td class="row-title">DHT:</td>
    <td>
      <a href="https://github.com/irungentoo/toxcore/blob/master/docs/updates/DHT.md">Home-made based of bittorrent DHT</a>
    </td>
    <td>
      <a href="https://github.com/savoirfairelinux/opendht">Home-made called OpenDHT</a>
    </td>
  </tr>
  <tr>
    <td class="row-title">Encryption:</td>
    <td>
      <a href="http://doc.libsodium.org/public-key_cryptography/index.html">libsodium public key infrastructure</a>
    </td>
    <td>
      <a href="http://www.gnutls.org/manual/html_node/X509-certificate-API.html">GnuTLS x509 Infrastructure</a>
    </td>
  </tr>
  <tr>
    <td class="row-title">Community:</td>
    <td><a href="https://github.com/irungentoo/toxcore">Hosted on Github</a></td>
    <td>Hosted on a <a href="https://gerrit-ring.savoirfairelinux.com/#/q/status:open">public gerrit instance</a>. Difficult to access, slow internet, dead mailing list.</td>
  </tr>
  <tr>
    <td class="row-title">Client Solution:</td>
    <td>
      Use as an embedded library.<br />
      <code>#include&nbsp;&lt;tox/tox.h&gt;</code>
    </td>
    <td>Run a system daemon and interact over d-bus</td>
  </tr>
  <tr>
    <td class="row-title">Contributors:</td>
    <td>
      <ul>
        <li><b>2064</b>&nbsp;irungentoo (4&nbsp;weeks&nbsp;ago)</li>
        <li><b>138</b>&nbsp;Maxim&nbsp;Biro (4&nbsp;months&nbsp;ago)</li>
        <li><b>116</b>&nbsp;Coren[m] (1&nbsp;year,&nbsp;9&nbsp;months&nbsp;ago)</li>
        <li><b>93</b>&nbsp;mannol (8&nbsp;months&nbsp;ago)</li>
        <li><b>78</b>&nbsp;Jfreegman (8&nbsp;weeks&nbsp;ago)</li>
        <li><b>65</b>&nbsp;Sean&nbsp;Qureshi (11&nbsp;months&nbsp;ago)</li>
        <li><b>41</b>&nbsp;notsecure (1&nbsp;year,&nbsp;1&nbsp;month&nbsp;ago)</li>
        <li><b>37</b>&nbsp;Astonex (2&nbsp;years,&nbsp;1&nbsp;month&nbsp;ago)</li>
        <li><b>36</b>&nbsp;charmlesscoin (2&nbsp;years,&nbsp;1&nbsp;month&nbsp;ago)</li>
        <li><b>27</b>&nbsp;slvr (2&nbsp;years,&nbsp;1&nbsp;month&nbsp;ago)</li>
      </ul>
    </td>
    <td>
      <ul>
        <li><b>3380</b>&nbsp;Tristan&nbsp;Matthews (7&nbsp;months&nbsp;ago)</li>
        <li><b>2311</b>&nbsp;Alexandre&nbsp;Savard (2&nbsp;years,&nbsp;10&nbsp;months&nbsp;ago)</li>
        <li><b>1814</b>&nbsp;Emmanuel&nbsp;Milou (1&nbsp;year&nbsp;ago)</li>
        <li><b>518</b>&nbsp;Rafaël&nbsp;Carré (11&nbsp;months&nbsp;ago)</li>
        <li><b>436</b>&nbsp;Adrien&nbsp;Béraud (11&nbsp;days&nbsp;ago)</li>
        <li><b>419</b>&nbsp;Guillaume&nbsp;Roguez (10&nbsp;Hours&nbsp;Ago)</li>
        <li><b>347</b>&nbsp;Emmanuel&nbsp;Lepage (5&nbsp;weeks&nbsp;ago)</li>
        <li><b>270</b>&nbsp;Julien&nbsp;Bonjean (5&nbsp;years&nbsp;ago)</li>
        <li><b>263</b>&nbsp;yanmorin (8&nbsp;years&nbsp;ago)</li>
        <li><b>203</b>&nbsp;jpbl (10&nbsp;years&nbsp;ago)</li>
      </ul>
    </td>
  </tr>
</table>

## Ring

The bulk of distributed nature of Ring is a fairly new endevour, where the bulk
of the work was performed during the last year.

Ring did start as the SFLphone project which has been ongoing since 2004. As
such it has a very long history.

The project was founded and is supported by a FOSS consultancy named
[Savoir-faire Linux](https://www.savoirfairelinux.com/en/).
