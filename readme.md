- [x] Untuk implementasi RTBH di IXP Manager, sebelumnya anda harus membuat Skinning di IXP Mangager terlebih dahulu.
Skinning ini dimaksudkan untuk menempatkan template yang sudah dimodifikasi. Dokumentasi mengenai skinning di IXP Manager di link berikut : https://docs.ixpmanager.org/features/skinning/

1. Copy file `/srv/ixpmanager/resources/views/api/v4/router/server/bird2/community-filtering-definitions.foil.php` ke folder target skinning yang sudah dibuat.
2. Edit File `$skinfolder/community-filtering-definitions.foil.php` tambahkan baris beriikut dibawah line # Informational prefixes :
```diff
    define IXP_BLACKHOLE = ( routeserverasn, 65535, 666 );
```
3. Copy `/srv/ixpmanager/resources/views/api/v4/router/server/bird2/neighbors.foil.php` ke folder target skinning
4. Edit `$skinfolder/neighbors.foil.php`, replace policy # Filter Small Prefix dengan 
```diff
    # Filter small prefixes & detection RTBH
<?php if( $t->router->protocol == 6 ): ?>
    if ( net ~ [ ::/0{<?= config( 'ixp.irrdb.min_v6_subnet_size', 48 ) == 128 ? 128 : config( 'ixp.irrdb.min_v6_subnet_size', 48 ) + 1 ?>,128} ] ) then {	
<?php else: ?>
    if ( net ~ [ 0.0.0.0/0{<?= config( 'ixp.irrdb.min_v4_subnet_size', 24 ) == 32 ? 32 : config( 'ixp.irrdb.min_v4_subnet_size', 24 ) + 1 ?>,32} ] ) then {
<?php endif; ?>
	if ( (65535, 666) ~ bgp_community ) then {
		bgp_large_community.add( IXP_BLACKHOLE );
		accept;
		} else {
			bgp_large_community.add( IXP_LC_FILTERED_PREFIX_LEN_TOO_LONG );
			accept;
		}
	}
```
5. Masih di file yang sama, edit juga pada bagian filter f_export_as?  . ganti nexthopnya dengan ip address blackhole
```diff
filter f_export_as<?= $int['autsys'] ?>
{

<?php
    // We allow per customer AS export code here which IXPs can define as skinned files.
    // For example, to solve a Facebook issue, INEX created the following:
    //     resources/skins/api/v4/router/server/bird2/f_export_as32934.foil.php
    echo $t->insertif( 'api/v4/router/server/bird2/f_export_as' . $int['autsys'] );
?>

<?php if( $t->router->protocol == 6 ): ?>
if ( (65535, 666) ~ bgp_community ) then {
	bgp_next_hop=::fff;
	accept;
	} else {
<?php else: ?>
if ( (65535, 666) ~ bgp_community ) then {
	bgp_next_hop=123.0.0.66;
	accept;
	} else {
<?php endif; ?>


    # we should strip our own communities which we used for the looking glass
    bgp_large_community.delete( [( routeserverasn, *, * )] );
    bgp_community.delete( [( routeserverasn, * )] );

    # default position is to accept:
    accept;
	}
}
```
6. Untuk menampilkan keterangan pada looking-glass edit file `/srv/ixpmanager/app/Utils/Foil/Extensions/Bird.php`
   tambahkan 
```diff
':65535:666'  => [ 'BLACKHOLE', 'info' ],
```
 di dalam `public static $BGPLCS = [ ... ]`
