# SoapClientGUS

This micro project provides a helper class for accessing Polish [REGON API](https://api.stat.gov.pl/Home/RegonApi).

GUS provides API based on:

- SOAP 1.2
- WS-Addressing
- Multipart messages
- HTTP header authentication

Sadly that is poorly supported by PHP's `SoapClient`. That's where this helper class comes handy.

## `SoapClientGUS` class

```
<?php
// SPDX-License-Identifier: Unlicense
class SoapClientGUS extends \SoapClient {
    protected $stream;

    function __construct(?string $wsdl, array $options = [])
    {
        $this->stream = stream_context_create();

        $options = array_merge([
            // Use SOAP 1.2 to avoid: Cannot process the message because the content type 'text/xml; charset=utf-8' was not the expected type 'multipart/related; type="application/xop+xml"'.
            'soap_version' => SOAP_1_2,
            // Custom stream to allow adding HTTP headers
            'stream_context' => $this->stream,
        ], $options);

        return parent::__construct($wsdl, $options);
    }

    public function __doRequest($request, $location, $action, $version, $one_way = null)
    {
        // GUS requires WS-Addressing
        $dom = new DomDocument('1.0');
        $dom->loadXML($request);
        $header = $dom->createElementNS('http://www.w3.org/2003/05/soap-envelope', 'Header');
        $header->appendChild($dom->createElementNS('http://www.w3.org/2005/08/addressing', 'Action', $action));
        $header->appendChild($dom->createElementNS('http://www.w3.org/2005/08/addressing', 'To', $location));
        $dom->documentElement->insertBefore($header, $dom->documentElement->firstChild);
        $request = $dom->saveXML();

        $response = parent::__doRequest($request, $location, $action, $version);

        // Get just an XML from the multipart message response (SoapClient doesn't support it)
        return preg_replace('/.*\n(<.*>).*/s', '\1', $response);
    }

    public function __setSid($sid)
    {
        stream_context_set_option($this->stream, [
            'http' => [
                'header' => 'sid: '. $sid,
            ]
        ]);
    }
}
```

## Example

```
<?php
// SPDX-License-Identifier: Unlicense
define('GUS_REGON_WSDL', 'https://wyszukiwarkaregontest.stat.gov.pl/wsBIR/wsdl/UslugaBIRzewnPubl-ver11-test.wsdl');
define('GUS_REGON_KEY', 'abcde12345abcde12345');

//define('GUS_REGON_WSDL', 'https://wyszukiwarkaregon.stat.gov.pl/wsBIR/wsdl/UslugaBIRzewnPubl-ver11-prod.wsdl');
//define('GUS_REGON_KEY', 'putyoursecretkeyhere');

try {
    $gus = new SoapClientGUS(GUS_REGON_WSDL);

    /* Sign in */

    $result = $gus->Zaloguj([
        'pKluczUzytkownika' => GUS_REGON_KEY,
    ]);
    if (empty($result->ZalogujResult)) {
        throw new Exception('Failed to login');
    }

    $sid = $result->ZalogujResult;
    $gus->__setSid($result->ZalogujResult);

    /* Search */

    $result = $gus->DaneSzukajPodmioty([
        'pParametryWyszukiwania' => [
            'Nip' => '5261040828',
        ],
    ]);
    var_dump($result->DaneSzukajPodmiotyResult);

    /* Report */

    $result = $gus->DanePobierzPelnyRaport([
        'pRegon' => '000331501',
    ]);
    var_dump($result->DanePobierzPelnyRaportResult);

    /* Sign out */

    $result = $gus->Wyloguj([
        'pIdentyfikatorSesji' => $sid,
    ]);
} catch (Exception $e) {
    echo $e->getMessage();
}
