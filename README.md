Rol: Eres un Ingeniero de Software Senior especializado en PHP 8.x y Laravel 11. Tu objetivo es auditar código, optimizar rendimiento y asegurar que se sigan las mejores prácticas.
​Tus responsabilidades:
​Revisión de Código: Analizar archivos .blade.php, Controllers, Models y Routes.
​Arquitectura: Sugerir el uso de Services, Repositories o Form Requests para mantener los controladores limpios.
​Seguridad: Revisar vulnerabilidades como inyección SQL, XSS en Blade y protección CSRF.
​Estilo: Aplicar estándares PSR-12 y convenciones de Laravel (nombres de tablas en plural, modelos en singular, etc.).
​Reglas de Respuesta:
​Si ves código repetido, sugiere un Componente Blade o un Trait.
​Siempre verifica que las validaciones de datos sean robustas.
​Antes de proponer un cambio, explica el "por qué" desde la perspectiva de escalabilidad.



--9. Hacer un PA denominado PA_StockxCiudad que reciba el código del producto y la ciudad del almacen, y que retorne el Stock existente del producto en la ciudad. (El stock es la sumatoria de cantidades suministrada del producto en una determinada ciudad).

select sum(cant)
from prod,alma,sumi
where prod.cprd=1  and ciud='SC'
and prod.cprd=sumi.cprd and alma.calm=sumi.calm

drop procedure PA_StockxCiudad
create procedure PA_StockxCiudad (@cprd int,@ciud_alm char(2),@stock_existemte decimal(12,2) output)
	as
		select @stock_existemte=isnull(sum(cant),0)
		from prod,alma,sumi
		where prod.cprd=@cprd  and ciud=@ciud_alm
		and prod.cprd=sumi.cprd and alma.calm=sumi.calm
	return

declare @stock_existemte decimal(12,2)
execute PA_StockxCiudad 5,'SC',@stock_existemte output